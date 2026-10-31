---
vendor: Arista
os: EOS
topic: OSPF
devices: vEOS、DCS-7050X/7060X/7280R/7500R 等 EOS 设备
keywords: [OSPF, router ospf, network area, router-id, passive-interface, ip ospf area, ip ospf cost, hello-interval, dead-interval, show ip ospf neighbor]
sources:
  - https://www.arista.com/en/um-eos/eos-open-shortest-path-first-version-2
  - https://www.arista.com/en/um-eos/eos-routing-protocols
  - https://www.arista.com/en/um-eos/eos-ipv4
verified: high
updated: 2026-10-30
---

# Arista · OSPF

> 适用于 EOS 的 OSPFv2（RFC 2328）。关键差异：**必须全局 `ip routing`**，否则进 `router ospf` 会直接提示 `! IP routing not enabled`；EOS 默认只跑一个 OSPFv2 进程（多实例需 multi-agent 模式）。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 启动进程 | `router ospf 1` | `(config)#` | 默认仅支持一个 OSPFv2 实例 |
| 配置 Router ID | `router-id 10.1.1.1` | `(config-router-ospf)#` | 未配则取 loopback，再取最高 IP |
| 宣告网段（CIDR） | `network 10.1.10.0/24 area 0` | `(config-router-ospf)#` | running-config 统一存为前缀写法 |
| 宣告网段（反掩码） | `network 10.1.10.0 0.0.0.255 area 0` | `(config-router-ospf)#` | 与上一条等价，host 位会被清零 |
| 接口级宣告 | `ip ospf area 0` | 接口视图 | 替代 `network`，配置更贴近接口 |
| 配置被动接口 | `passive-interface Vlan10` | `(config-router-ospf)#` | 只发 LSA 不发 Hello，接主机网段必配 |
| 区域类型 | `area 45 stub` / `area 1 nssa` | `(config-router-ospf)#` | `area 1 nssa no-summary` 可挡 Type-3 |
| 区域汇总 | `area 5 range 10.1.25.64 0.0.0.192` | `(config-router-ospf)#` | ABR 上聚合并可抑制明细 |
| 重分发直连 | `redistribute connected` | `(config-router-ospf)#` | 全部直连变 Type-5 外部路由 |
| 重分发静态 | `redistribute static` | `(config-router-ospf)#` | EOS 不支持只重分发单条静态路由 |
| 下发默认路由 | `default-information originate` | `(config-router-ospf)#` | 可选参数随版本，以官方手册为准 |
| 过滤入向路由 | `distribute-list prefix-list PL1 in` | `(config-router-ospf)#` | SPF 之后过滤，LSA 仍会继续传 |
| 调整管理距离 | `distance ospf intra-area 95` | `(config-router-ospf)#` | 三类路由默认均为 **110** |
| 邻接日志 | `log-adjacency-changes` | `(config-router-ospf)#` | 加 `detail` 记录全部 LS 变化 |
| 关闭 OSPF | `shutdown` | `(config-router-ospf)#` | 保留配置仅停协议，`no shutdown` 恢复 |
| 接口 cost | `ip ospf cost 15` | 接口视图 | 默认 **10**，与带宽无关需手工调 |
| Hello/Dead 定时器 | `ip ospf hello-interval 30` / `ip ospf dead-interval 120` | 接口视图 | 默认 10 / 40，两端必须一致 |
| DR 优先级 | `ip ospf priority 15` | 接口视图 | 默认 **1**，配 0 则不参与 DR/BDR 选举 |
| 简单口令认证 | `ip ospf authentication-key 0 code123` | 接口视图 | 需先 `ip ospf authentication` |
| MD5 认证 | `ip ospf message-digest-key 23 md5 0 code123` | 接口视图 | 需先 `ip ospf authentication message-digest` |
| 网络类型 | `ip ospf network point-to-point` | 接口视图 | 点对点链路免 DR 选举、收敛更快 |
| 查看邻居 | `show ip ospf neighbor` | `>` 特权 EXEC | 关注 State=FULL 与 Dead Time |
| 查看进程/区域 | `show ip ospf` | `>` 特权 EXEC | 区域、接口数、SPF 次数、汇总条目 |
| 查看接口参数 | `show ip ospf interface` | `>` 特权 EXEC | cost、定时器、认证、区域 |
| 查看 LSDB | `show ip ospf database` | `>` 特权 EXEC | 按 LSA 类型排查"有邻居无路由" |

## 典型示例

```text
switch# configure terminal
switch(config)# ip routing                              // ★ 前置条件，默认关闭
switch(config)# router ospf 1
switch(config-router-ospf)# router-id 10.255.0.1
switch(config-router-ospf)# network 10.10.10.0/24 area 0
switch(config-router-ospf)# network 10.0.0.0/30 area 0
switch(config-router-ospf)# passive-interface Vlan20    // 用户网段不发 Hello
switch(config-router-ospf)# redistribute connected
switch(config-router-ospf)# log-adjacency-changes
switch(config-router-ospf)# exit
switch(config)# interface Vlan20
switch(config-if-Vl20)# ip ospf cost 15
switch(config-if-Vl20)# exit
switch(config)# interface Ethernet49                    // 点对点上联
switch(config-if-Et49)# no switchport
switch(config-if-Et49)# ip address 10.0.0.1/30
switch(config-if-Et49)# ip ospf network point-to-point
switch(config-if-Et49)# ip ospf authentication message-digest
switch(config-if-Et49)# ip ospf message-digest-key 23 md5 0 code123
switch(config-if-Et49)# exit
switch(config)# end
switch# copy running-config startup-config              // ★ 不保存则重启丢失
switch# show ip ospf neighbor
switch# show ip ospf interface
switch# show ip route
```

## 陷阱与版本差异

- **OSPF 起不来先查 `ip routing`**：官方明确说明未开启时进入 `router ospf` 会打印 `! IP routing not enabled`，此时邻居永远起不来。
- EOS 默认只允许**一个 OSPFv2 实例**，重复 `router ospf <另一个进程号>` 会报错；多实例仅在 multi-agent 路由模式下支持，且全部实例须在 default VRF。
- 默认 OSPF 管理距离 **110**，静态路由为 **1**——所以想用 OSPF 覆盖某条静态路由，必须给静态配更大 distance 或直接删静态。
- `network` 里的地址写 CIDR 或"地址+反掩码"均可，EOS 会把 host 位清零（`1.2.3.4/24` → `1.2.3.0/24`）；写成掩码形式 `255.255.255.0` 会被拒。
- 接口 `ip ospf cost` 默认固定为 **10**、不随带宽变化（与 Cisco 按带宽算 cost 不同），跨厂商混合组网时 cost 需要统一规划。
- Hello/Dead 不一致（默认 10/40）表现为邻居卡在 INIT/EX2START；`ip ospf dead-interval` 必须 ≥ 两端 hello 间隔。
- 同前缀多实例/多来源选路顺序为 `O > O IA > N1 > N2 > E1 > E2`，同类型再比下一跳 IP，且**不生成 ECMP**（官方特别说明）。

## 高危操作 ⚠️

- ⚠️ 在 `router ospf` 下执行 `shutdown` 会立刻拆掉全部邻居、清除 OSPF 路由，依赖该协议的业务瞬间中断；先确认是否有静态/备份路径。
- ⚠️ 运行中修改 `distance ospf` 官方明确提示会**导致 OSPF 链路抖动**，必须在维护窗口操作并准备回退。
- ⚠️ 误删 `passive-interface` 或错误宣告大网段（如 `network 0.0.0.0/0 area 0`）会把不该进 OSPF 的接口拉进来，引发邻居震荡与路由泄漏。
- ⚠️ 修改 `ip ospf authentication` 相关配置需两端同时变更；单端先改会立即拆邻居，建议先配好新 key 再切换。
- 变更顺序：`show running-config` + `show ip ospf neighbor` 留存基线 → 变更 → `show ip ospf neighbor` / `show ip route` 验证 → 异常时 `shutdown` 该进程或回退配置 → 确认后 `copy running-config startup-config`。

## 参考来源

1. [EOS User Manual · Open Shortest Path First – Version 2（arista.com）](https://www.arista.com/en/um-eos/eos-open-shortest-path-first-version-2)
2. [EOS User Manual · Routing Protocols（arista.com）](https://www.arista.com/en/um-eos/eos-routing-protocols)
3. [EOS User Manual · IPv4 · Route Management（arista.com）](https://www.arista.com/en/um-eos/eos-ipv4)
