# 用户态 VM 网络：基于 HCO + KubeVirt 的 vhost-user 接入设计

> 状态：**设计 / 开发指南**（指导后续实现，非"已完成"记录）
> 适用仓库：本仓库（fork 自 `kubevirt/hyperconverged-cluster-operator`）
> 定位：在 **HCO 基石**之上，把 KubeVirt VM 接入我们已自研并真机验证的**用户态数据面底座**（userspace-cni + nv-ipam + bird-k8s / bird-vpp），全程绕开内核。

------

## 0. 一句话目标

让 KubeVirt 的 VM 通过 **vhost-user** 接到**节点 VPP（bird-vpp）**的 bridge-domain，以 bird-k8s 编程的 **BVI** 为 L3 网关，由 bird-k8s 把**本节点聚合 CIDR** 通过 BGP 通告给 fabric——**与我们已验证的 Pod 用户态路径同底座、同控制面**，差的只是 KubeVirt 侧一个 **vhost-user network binding 插件（sidecar）**。

------

## 1. 背景与定位：内核态用 HCO 原生，用户态由我们扩展

| 平面 | 提供方 | 我们的角色 |
|---|---|---|
| **控制面 / 编排**（KubeVirt + CDI + 升级 + 绑定注册）| **HCO**（本仓库）| 踩在它上面 |
| **内核态网络**（bridge / macvtap / OVS-kernel）| HCO 自带的 **CNAO** | 原生支持，**不是我们的目标** |
| **用户态网络**（memif / vhost-user → VPP）| **userspace-cni + bird-k8s + nv-ipam**（我们）| 扩展开发 |
| **VM↔用户态口的桥接**（注入 libvirt domain XML）| 无人提供 | **本设计的核心交付物：binding sidecar** |

**为什么 VM 线踩 HCO 更合适**（前期调研结论）：
- HCO 把 KubeVirt + **CDI**（VM 磁盘导入/DataVolume，VM 比容器多出来的一摊）+ SSP + 升级编排一把管；裸装 KubeVirt 还得自己拼 CDI。
- HCO 原生暴露 **网络绑定插件注册入口**（`spec.networking.networkBinding`，原样透传到 KubeVirt），我们的 sidecar 在此声明式注册。
- HCO 提供 **jsonpatch 逃生舱**，可开 KubeVirt 未在 HCO 暴露的 feature gate（如 `Sidecar`）、补 hugepages/NUMA。
- HCO 是 **OpenShift Virtualization（CNV）的上游**——踩它=对齐生产级虚拟化基线。

> **务必分清**：HCO 抬高的是**控制面**基石；**数据面仍是 userspace-cni + bird-k8s + 本设计的 sidecar**。HCO/CNAO/KubeVirt 谁都不提供 vhost-user 数据面。

------

## 2. 上游现状与先例（决定路线的关键）

### 2.1 原生 vhost-user 尝试（2020–2022）全部烂尾

| 项目 | 结局 | 备注 |
|---|---|---|
| `kubevirt#3405`（issue）| **rotten 自动关闭** | 用户用 OVS-DPDK + Multus + **userspace-cni**，VM 接不上，报 `failed to get a link for interface: net1 / Link not found`——KubeVirt 对 secondary 网卡默认走内核绑定（找 netlink link），而 vhost-user 口在内核里没有 link |
| `kubevirt#3208`（PR）| **CLOSED，never merged** | 一个完整实现（+7616/-928）：加 `InterfaceVhostuser` API、virtwrap 生成 vhostuser domain XML、NUMA `memAccess='shared'`、用 **app-netutil** 读 userspace-cni annotation 拿 socket、改 libvirt 镜像。太大，要求拆小 PR |
| `#3843 / #3888 / #3481 / #5662`（拆分 PR）| **全部 CLOSED 未合并** | 拆分后失去 review 动力，全部 lifecycle/rotten |
| `kubevirtci#365` / `libvirt#56` | 关闭 | 测试基础设施 / qemu 用户组改动 |
| `kubevirt#8873`（issue）| **rotten 关闭** | 2022 年 NFV/Edge（如 Palo Alto 防火墙）再次提需求，列任务清单，全 `[ ]` 未打勾 |

**教训**：把 vhost-user 做成 KubeVirt **核心 API/代码**的路线，社区走不通。

### 2.2 但框架化路线成功了（2024–2026）

KubeVirt 引入 **network binding plugin** 框架，vhost-user 改为**树外（out-of-tree）插件**落地：

- **框架**：`spec.configuration.network.binding.<name>`（`sidecarImage` + `domainAttachmentType` + `downwardAPI` + `computeResourceOverhead`），**v1.4 Beta → v1.5 GA**（PR #17438）。
- **vhost-user 已合入**：`kubevirt#14539`（2025-05 合并）"Enable vhost-user" 给 **passt binding** 加了 vhost-user 模式 + shared MemoryBacking；后续 #14832/#15019/#16475/#16820 持续完善，**passt 绑定（含 vhost-user）已 GA（VEP #21）**。
- **device-info 投递机制**：`downwardAPI: device-info` → virt-controller 写 `kubevirt.io/network-info` annotation → virt-launcher 投影到 `/etc/podinfo/network-info` → sidecar 读 `DeviceInfo.Vhost.SocketPath`（来自 k8snetworkplumbingwg device-info-spec）。
- **活参考实现**：`cmd/sidecars/network-passt-binding/`（已做 vhost-user）、`cmd/sidecars/network-slirp-binding/`。

**结论**：我们要做的 vhost-user binding，**有了完整现代框架 + 一个已实现 vhost-user 的参考（passt）**，且 #3208 趟过的所有技术坑（XML 形态、shared-mem NUMA、socket 权限）都可直接借鉴——但**不需要改 KubeVirt 核心**，做成树外 sidecar 即可。这是当年 #3208 没有的有利条件。

------

## 3. Nova 对照：为什么需要 sidecar，它替代了哪一层

OpenStack 不是"原生支持 vhost-user"，而是把工作拆给三层协作（调研自 `openstack-nova`）：

```
Neutron ML2 agent(OVS-DPDK/VPP)  → 建 vhost-user socket，把 path+mode 写进 binding:vif_details
os-vif(插件库)                    → plug：校验/接入转发
Nova(libvirt 驱动)               → 读 vif_details，生成 <interface type='vhostuser'> domain XML
QEMU                             → 按 XML 连 socket（通常 OVS=server，QEMU=client）
```
- socket **不是 Nova 建的**（是 Neutron ML2 agent 建的，`nova/network/model.py` 的 `VIF_DETAILS_VHOSTUSER_SOCKET/MODE`）。
- Nova `designer.set_vif_host_backend_vhostuser_config` 生成 XML。
- **关键印证**：`nova/virt/libvirt/driver.py` 明确——vhost-user 要求 guest 内存是 **大页/文件 + `memAccess='shared'`**，Nova 请求 hugepages 时**无条件**给 NUMA cell 打 `memAccess="shared"`。这与 #3208 和本设计完全一致：**hugepages + shared mem 是 vhost-user 协议本身的硬要求，不是某个实现的麻烦。**

**映射到 K8s 用户态（我们）**：

| 职责 | OpenStack | 我们 | 状态 |
|---|---|---|---|
| 建 vhost-user socket 到转发器 | Neutron ML2 agent | **userspace-cni** | ✅ 已有 |
| 插件化 plug | os-vif | （CNI 内/sidecar 内）| ✅/部分 |
| 注入 libvirt vhostuser XML | Nova libvirt 驱动 | **本设计的 binding sidecar** | ⚠️ 待开发 |
| 编排/装载 | （各自）| **HCO** | ✅ 基石 |

> 即：**Neutron+os-vif 那两层我们用 userspace-cni 顶上了**；缺的只是"Nova libvirt 驱动注入 XML"那一层——本设计的 sidecar。

------

## 4. 底座成熟度盘点（决定走哪条用户态路径）

### 4.1 bird-vpp / bird-k8s（控制面 + L3）——**已真机验证**

- **bird-vpp**：节点 VPP 数据面（4k 页可无 hugepage 跑；vhost-user 需 hugepage，见 §6）。
- **bird-k8s**：`vpp + bird + controller` 三容器；`PerNodeCIDRProvider`（nv-ipam / whereabouts 双 adapter）→ 起源本节点聚合段 + 编程 **BD BVI**（网关）+ BGP 通告；`vpp.Watcher` 在 VPP 重启后**秒级重建 BVI**。
- 已验证：v4/v6 双栈、100 Pod 规模、暴力 churn 无残留、固定 IP、南北向 ping 通 fabric。
- **L3 模型绑定在 VPP 引擎**（BD + BVI + redistribute static）。

### 4.2 userspace-cni——分路径成熟度不同

| 引擎/类型 | 实现 | 成熟度 |
|---|---|---|
| **VPP / memif**（容器）| 完整 | ✅ **生产级**：功能测试 + restore daemon（VPP 重启自愈、周期 reconcile、孤儿 socket sweep）+ 本会话 100 规模/暴力验证 |
| **VPP / vhostuser** | 完整 | ✅ 走同一 VPP 引擎；VM 用这条 |
| **OVS-DPDK / vhostuser** | 控制面完整（`cniovs/ovsdb.go` 直连 libovsdb，`datapath_type=netdev`，server+client 双模，VLAN/QoS/OVN）| ⚠️ **未测脚手架**：单测仅 JSON/wire 解析（不测 createVhostPort）、CI e2e 脚本存在但 `run_all` 注释未启用、**restore daemon 明确不支持 OVS**（`desired.go` 非 vpp 引擎静默跳过）、**仅二层（netType=interface 被拒），无 L3/BVI** |
| OVS / memif | 不支持 | memif 是 VPP 专属 |
| **IPAM 网关透传** | 引擎无关 | ✅ 我们修的 `IPResult` 网关透传 + CNI-arg 容忍补丁，OVS/VPP 都生效 |

### 4.3 路径抉择：**VM 走 VPP vhost-user，不走 OVS-DPDK**

| | VPP vhost-user（选）| OVS-DPDK vhost-user |
|---|---|---|
| L3 / per-node-CIDR / BVI / BGP 通告 | ✅ bird-k8s 全套 | ❌ 仅二层，与我们的 L3 通告模型不搭 |
| restore / 规模 / 重启自愈 | ✅ 已验证 | ❌ 无 restore，未测 |
| 与已验证 Pod 路径同底座 | ✅ | ✗ |
| 落地成本 | 低（复用全套）| 高（要补单测/CI/restore/L3）|

> **决策**：VM 用户态走 **userspace-cni（VPP 引擎）vhost-user**。OVS-DPDK 留作未来"必须 OVS 生态"的备选，前提是先补它的测试/CI/restore 三块。

------

## 5. 目标架构

```
┌ Node (e.g. network1) ───────────────────────────────────────────────┐
│ virt-launcher Pod                                                    │
│   ┌─ compute(QEMU/KVM) ── virtio-net ─┐                              │
│   │                                   │ vhost-user socket(用户态)    │
│   └─ vhost-user binding sidecar ◄─────┘  (本设计交付：OnDefineDomain  │
│        读 /etc/podinfo/network-info     注入 <interface vhostuser>)   │
│                                   ▼                                   │
│   userspace-cni(VPP 引擎) 建 vhost-user 口 ──► 节点 VPP               │
│                                              BD100 ── BVI(loop1)      │
│   nv-ipam CIDRPool ──► IP+网关(权威)          │   10.66.0.1/24        │
│   bird-k8s: 起源 /24+/64 + 编程 BVI + 通告 ──┴─► 上联 → fabric        │
│   vpp.Watcher: VPP 重启 → 秒级重建 BVI                                │
└──────────────────────────────────────────────────────────────────────┘
```

数据通路：**QEMU virtio-net ↔ vhost-user socket ↔ 节点 VPP BD100 ↔ BVI(L3 网关) ↔ 上联 ↔ fabric**，全程用户态、绕内核。控制面（IP/网关来源、聚合通告、BVI 自愈）**与已验证的 Pod 路径完全一致**。

------

## 6. 核心交付物：vhost-user network binding sidecar

### 6.1 注册（HCO → KubeVirt）

HCO 原生暴露 `spec.networking.networkBinding`（`api/v1/hyperconverged_types.go` 的 `NetworkBinding map[string]InterfaceBindingPlugin`，原样 clone 进 KubeVirt CR）：

```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: kubevirt-hyperconverged
  annotations:
    # 'Sidecar' 是 KubeVirt Alpha gate，HCO 未在 curated 列表暴露 → 用 jsonpatch 逃生舱开启
    kubevirt.kubevirt.io/jsonpatch: |
      [{"op":"add","path":"/spec/configuration/developerConfiguration/featureGates/-","value":"Sidecar"}]
spec:
  networking:
    networkBinding:
      vhostuser:
        sidecarImage: ghcr.io/fivetime/kubevirt-vhostuser-binding:latest
        downwardAPI: device-info            # ★ 把 multus device-info(含 socket) 投给 sidecar
        computeResourceOverhead:
          requests:
            memory: 128Mi
        # domainAttachmentType: 留空——本插件由 sidecar 直接注入 vhostuser 设备，不用 tap/managedTap
        # migration: 见 §9（per-node-CIDR 下热迁移受限）
```

> 注：`featureGates/-` 形式追加，避免覆盖 HCO 已注入的强制 gate。jsonpatch 注解键：`kubevirt.kubevirt.io/jsonpatch`（另有 CDI/CNAO/SSP 对应键）。

### 6.2 sidecar 契约（基于现代框架，参考 `network-passt-binding`）

- 监听 unix socket：`/var/run/kubevirt-hooks/vhostuser.sock`。
- 注册 gRPC `Info` 服务：name=`network-vhostuser-binding`，HookPoint=`OnDefineDomain`，version=`v1alpha2`（或 v1alpha3，含 `Shutdown`）。
- 实现 `OnDefineDomain(domainXML, vmi) → domainXML`：读 device-info → 解析 socket → 注入/改写 vhostuser 接口。

**main.go 骨架**（仿 `cmd/sidecars/network-passt-binding/main.go`）：
```go
const hookSocket = "vhostuser.sock"
func main() {
    socketPath := filepath.Join(hooks.HookSocketsSharedDirectory, hookSocket) // /var/run/kubevirt-hooks/
    lis, _ := net.Listen("unix", socketPath); defer os.Remove(socketPath)
    s := grpc.NewServer()
    hooksInfo.RegisterInfoServer(s, srv.InfoServer{Version: "v1alpha2"})
    hooksV1alpha2.RegisterCallbacksServer(s, srv.V1alpha2Server{})
    s.Serve(lis)
}
```

**OnDefineDomain（server）**：unmarshal VMI → 找到 `binding.name=vhostuser` 的 interface → 读 `/etc/podinfo/network-info`（`downwardapi.NetworkInfo`，按 network 名取 `DeviceInfo.Vhost.SocketPath`）→ unmarshal domain XML → 改写该接口为 vhostuser → marshal 返回。

### 6.3 注入的 libvirt domain XML（借鉴 #3208 + passt）

```xml
<interface type='vhostuser'>
  <mac address='52:54:00:..'/>
  <source type='unix' path='/var/lib/cni/usrspcni/<sock>' mode='server'/>   <!-- 见 §6.5 mode -->
  <model type='virtio'/>
  <driver rx_queue_size='1024' tx_queue_size='1024'/>
  <alias name='ua-vhostuser-<net>'/>
</interface>
```
对应 `pkg/virt-launcher/virtwrap/api` 的 `Interface{Type:"vhostuser", Source:{Type:"unix",Path,Mode}, Model, Driver:{RxQueueSize,TxQueueSize}}`。

### 6.4 socket 路径来源——一个必须解决的 gap ⚠️

现代框架走 **device-info downwardAPI**：sidecar 读 `/etc/podinfo/network-info` 里的 `DeviceInfo.Vhost.SocketPath`。但 **device-info 要由 CNI 写进 multus `network-status` annotation**（k8snetworkplumbingwg device-info-spec，type=`vhost-user`）。

调研发现：**userspace-cni 目前把 socket 写进自己的 `userspace/configuration-data` annotation（自定义格式），不一定输出标准 DeviceInfo。** 两条路：

| 方案 | 做法 | 取舍 |
|---|---|---|
| **A（推荐）** | **扩展 userspace-cni 输出标准 DeviceInfo**（network-status 里 `device-info{type:"vhost-user", vhost:{path,mode}}`）| sidecar 走 KubeVirt 标准 device-info 路径，与 passt 同款，最可维护；改动在 userspace-cni（我们自己的仓库）|
| B | **sidecar 直接读 `userspace/configuration-data` annotation**（复用我们已有的，仿 #3208 的 app-netutil）| 不改 userspace-cni，但偏离标准框架；sidecar 需自己解析我们的私有格式 |

> 建议 **A**：在 userspace-cni 的 vhostuser 路径补一段"按 device-info-spec 写 network-status"，让整条链落在 KubeVirt 官方机制上。B 作为快速验证的临时手段。

### 6.5 shared memory / hugepages / NUMA（vhost-user 硬要求）

#3208 与 Nova 双重印证：**vhost-user 要求 guest 内存文件/大页 + `memAccess='shared'`**，否则节点 VPP 进程读不到 guest 共享内存。

- VM spec 必须 `spec.domain.memory.hugepages.pageSize`（如 `2Mi`）+ `spec.domain.cpu.dedicatedCpuPlacement: true`。
- KubeVirt 在 VMI 有 hugepages 时会给 NUMA cell 打 `memAccess='shared'`（#3208 在核心做了，现代 passt 路径用 `MemoryBacking` shared/memfd 达到同效）。
- **节点前置**：节点必须预留 hugepages（我们的 Pod 用 4k 页绕开，**VM 这条绕不开**）。这与"无 hugepage 的 bird-vpp 4k 页模式"不同，需评估节点 VPP 的 buffer 是否也上 hugepages。

### 6.6 socket 权限——#3208 踩过的真实坑

QEMU 进程（libvirt 下）要能读写 userspace-cni 建的 vhost-user socket。#3208 的处理：libvirt `qemu.conf` 加 `group = "hugetlbfs"`，socket 目录 `chown qemu:hugetlbfs`、`chmod g+r`。
- 我们的 sidecar/CNI 落地时需确保 **socket 的 uid/gid 让 virt-launcher 的 qemu 可访问**（共享目录 + 组权限）。这是容易漏的运行时坑，开发时优先验证。

------

## 7. VM 定义与 guest 取 IP

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: vm-userspace-1
  namespace: default
  annotations:
    # 固定 IP（可选）：NAD 需 capabilities.ips；双族固定依赖 userspace-cni 的 CNI-arg 容忍补丁
    k8s.v1.cni.cncf.io/networks: '[{"name":"vm-userspace","ips":["10.66.0.200","fc00:66::200"]}]'
spec:
  runStrategy: Always
  template:
    spec:
      domain:
        cpu: { cores: 2, dedicatedCpuPlacement: true }     # ★
        memory: { guest: 2Gi, hugepages: { pageSize: "2Mi" } }  # ★
        devices:
          disks:
          - { name: root, disk: { bus: virtio } }
          - { name: cloudinit, disk: { bus: virtio } }
          interfaces:
          - name: net1
            binding: { name: vhostuser }                   # ★ 引用 §6.1 注册的插件
      networks:
      - name: net1
        multus: { networkName: vm-userspace }              # userspace(VPP)+nv-ipam 的 NAD
      volumes:
      - name: root
        containerDisk: { image: quay.io/kubevirt/fedora-with-test-tooling-container-disk:latest }
      - name: cloudinit
        cloudInitNoCloud:
          networkData: |                                   # ★ guest 自己配 IP，值对齐 nv-ipam，网关=BVI
            version: 2
            ethernets:
              eth0:
                addresses: [ 10.66.0.200/24, fc00:66::200/64 ]
                gateway4: 10.66.0.1
                routes: [ { to: "::/0", via: "fc00:66::1" } ]
```

VM 用的 NAD（与 Pod 同构，`iftype: vhostuser`，IPAM=nv-ipam 双池=双栈）：
```json
{ "type":"userspace","name":"vm-userspace",
  "host":{"engine":"vpp","iftype":"vhostuser","netType":"bridge",
          "vhost":{"mode":"client"},"bridge":{"bridgeName":"100"}},
  "container":{"engine":"vpp","iftype":"vhostuser","netType":"interface","vhost":{"mode":"server"}},
  "ipam":{"type":"nv-ipam","poolName":"pod-pool,pod-pool-v6","poolType":"cidrpool"} }
```
> **guest 要手配 IP**：容器是 CNI 写进 netns；VM 的 guest OS 得自己拿。用 cloud-init `networkData` 静态配置（值对齐 nv-ipam 分配，网关=BVI）。也可 guest 内 DHCP，但需额外 DHCP 服务，静态最简。

------

## 8. 固定 IP（无 Service、按 IP 直连）

- NAD 加 `"capabilities": {"ips": true}`；VM 注解 `k8s.v1.cni.cncf.io/networks` 带 `ips`（§7）。nv-ipam 把地址钉给该口，重建/迁回本节点不变；cloud-init 写同样地址。
- **双族固定**依赖 userspace-cni 的 **CNI-arg 容忍补丁**（已合入：把 `k8sArgs.IP` 改成宽容字符串，否则 `IP=<v4>,<v6>` 逗号会让 GetPod 失败、报 `pod is nil`）。

------

## 9. 热迁移：诚实的限制

我们的 **per-node-CIDR 聚合模型**下，IP 属于"节点的段"（node1=`10.66.0.0/24`，node2=`10.66.1.0/24`…）。VM 迁到别的节点：
- **IP 改变**（从目标节点段重新分配），破坏"IP 不变"；或
- **保持 IP** → 需单独通告 `/32`（DC 路由爆炸，已否决）或上 **EVPN/VXLAN 二层延伸（后续版本）**。

**本设计定位**：固定 IP、钉在节点上的 VM（无 Service、直连）。**IP 保持的跨节点热迁移留给后续 EVPN。** 此外 vhost-user binding 能否 live-migrate 还取决于 sidecar 的 `binding.migration` 与 network-info 在迁移前就绪（参考 passt 的 `link-refresh` 与 PR #15663）。

------

## 10. 开发路线图（里程碑）

| 阶段 | 目标 | 验收 |
|---|---|---|
| **M0 基线** | HCO 装好 KubeVirt+CDI；节点配 hugepages；开 `Sidecar` gate（jsonpatch）| `kubectl get kv` Available；节点 hugepages>0 |
| **M1 参考验证** | 先用上游 **passt binding** 跑通一个 vhost-user VM（验证框架/hugepages/权限链路）| passt VM 起来、有 vhostuser 设备 |
| **M2 sidecar MVP** | 自研 `kubevirt-vhostuser-binding` sidecar（OnDefineDomain 注入 vhostuser XML）；socket 来源先走 **方案 B**（读 userspace-cni configuration-data）快速打通 | VM 接上 userspace-cni(VPP) vhost-user 口，节点 VPP 见到该口在 BD100 |
| **M3 标准化 device-info** | 扩展 **userspace-cni 输出标准 DeviceInfo**（方案 A），sidecar 改走 `downwardAPI: device-info` | sidecar 不再依赖私有格式；与 passt 同款路径 |
| **M4 L3 闭环** | bird-k8s 通告该 VM 段；guest cloud-init 配 IP+网关；ping BVI 与 fabric | VM ping 通 `10.66.0.1` / fabric（南北向）|
| **M5 固定 IP + 双栈** | `capabilities.ips` 固定 v4+v6；nv-ipam 双池 | 重建后 IP 不变；v4/v6 都通 |
| **M6 规模/自愈** | 多 VM；杀节点 VPP 验 BVI 自愈对 VM 的影响；socket 权限/重启边界 | 评估 VM 在 VPP 重启后的恢复行为（注意：restore daemon 当前面向 memif/Pod）|
| **M7（远期）** | EVPN/VXLAN → IP 保持热迁移 | 跨节点迁移 IP 不变 |

> 当前底座状态：M0 的 bird-k8s/nv-ipam/userspace-cni(VPP, Pod) 侧**已真机验证**；本设计从 **M1/M2** 起步。

------

## 11. 风险与未决问题

1. **device-info 输出**（§6.4）：userspace-cni 是否/如何输出标准 DeviceInfo——M3 的前提；先用方案 B 不阻塞 M2。
2. **socket 权限**（§6.6）：qemu uid/gid vs userspace-cni socket 属主——M2 优先验证，易踩。
3. **hugepages 与 bird-vpp 模式冲突**：节点 VPP 当前 4k 页跑；VM vhost-user 要 hugepages——需确认节点 VPP buffer 与 VM hugepages 共存策略。
4. **restore/自愈对 VM 的覆盖**：我们的 restore daemon 面向 memif/Pod（`desired.go` 跳过非 vpp/非 memif）；VM 的 vhost-user 口在节点 VPP 重启后的重建路径需单独评估（M6）。
5. **热迁移**（§9）：本版本不保 IP；EVPN 远期。
6. **sidecar 版本面**：hook v1alpha2 vs v1alpha3（含 Shutdown）；跟随上游 passt 的选择。

------

## 12. 参考

**上游 issue/PR**
- 原生尝试（烂尾）：`kubevirt/kubevirt#3405`、`#3208`、`#3843`、`#3888`、`#3481`、`#5662`、`#8873`；`kubevirt/kubevirtci#365`、`kubevirt/libvirt#56`
- 框架化成功：`kubevirt/kubevirt#14539`(passt vhost-user)、`#15663`(migration network-info)、`#17438`(binding plugins GA)；VEP #21（Passt GA）、VEP #40（vhostuser sidecar 共享卷）

**KubeVirt 源码（参考实现 / 契约）**
- `cmd/sidecars/network-passt-binding/`（main.go / server / domain configurator / callback）
- `pkg/hooks/v1alpha2`、`v1alpha3`（OnDefineDomain proto）；`pkg/hooks/hooks.go`（HookSidecar，`/var/run/kubevirt-hooks`）
- `pkg/network/downwardapi/`（NetworkInfo，`/etc/podinfo/network-info`，`DeviceInfo.Vhost.SocketPath`）
- `pkg/network/netbinding/netbinding.go`（按 `binding.name` 注入 sidecar）
- `staging/src/kubevirt.io/api/core/v1/types.go`（`InterfaceBindingPlugin`）

**Nova 对照（openstack-nova）**
- `nova/network/model.py`（`VIF_TYPE_VHOSTUSER`、`VIF_DETAILS_VHOSTUSER_*`）
- `nova/virt/libvirt/designer.py`（`set_vif_host_backend_vhostuser_config`）
- `nova/virt/libvirt/driver.py`（vhost-user 要求 `memAccess='shared'`）

**HCO（本仓库）**
- `api/v1/hyperconverged_types.go`（`NetworkingConfig.NetworkBinding`）
- `controllers/handlers/kubevirt.go`（透传到 KubeVirt CR）
- `controllers/common/consts.go`（jsonpatch 注解键）

**我们的底座**
- userspace-cni：`cnivpp/`(memif/vhostuser, VPP)、`cniovs/`(OVS-DPDK, 未测)、`pkg/daemon/`(restore, VPP-only)、`userspace/cni/cni.go`(IPResult 网关透传 + CNI-arg 容忍补丁)
- bird-k8s：`internal/podnet`(PerNodeCIDRProvider)、`internal/vpp`(EnsureBVI / Watcher)、`internal/bird`(渲染)、nv-ipam/whereabouts adapter
