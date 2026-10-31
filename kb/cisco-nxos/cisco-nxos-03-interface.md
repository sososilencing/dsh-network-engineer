---
vendor: 思科（Cisco）
os: NX-OS
topic: 接口配置
devices: Nexus 3000/5000/7000/9000 系列（含 9300、9500），Nexus 9000v
keywords: [接口, no switchport, ip address 斜线掩码, interface loopback, mgmt0, SVI, 子接口, encapsulation dot1q, mtu 9216, show ip interface brief]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_0100.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_static_routing.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · 接口配置

> 适用于 Nexus 的物理口三层化、SVI、Loopback、子接口与 mgmt0 带外管理口。
> 与 IOS 最大差异：**IP 地址一律写成 `A.B.C.D/前缀长度`**，没有独立的子网掩码字段。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进物理接口 | `interface Ethernet1/1` | `switch(config)#` | NX-OS 写 `Ethernet1/1`（简写 `e1/1`），没有 `GigabitEthernet0/0/1` 那种三段式 |
| 口描述 | `description To-CORE-VPC` | `switch(config-if)#` | 建议统一写对端设备+用途 |
| 切三层口 | `no switchport` | `switch(config-if)#` | 平台默认二层时必做；此操作会清空该口二层配置 |
| 配 IP | `ip address 10.1.1.1/24` | `switch(config-if)#` | **斜线前缀**；IOS 的 `ip address A mask` 写法不支持 |
| 配 IPv6 | `ipv6 address 2001:db8::1/64` | `switch(config-if)#` | 需 `feature ipv6`（随版本，以官方手册为准） |
| 创 SVI | `interface vlan 10` | `switch(config)#` | 前置 `feature interface-vlan`，否则视图进不去 |
| SVI 配 IP | `ip address 10.10.10.1/24` | `switch(config-if)#` | SVI 要 up，需该 VLAN 至少有端口 STP forwarding（autostate） |
| 环回口 | `interface loopback0` + `ip address 10.255.0.1/32` | `switch(config)#` | 常用作 router-id、BGP/OSPF 源；天然 always up |
| 带外管理口 | `interface mgmt0` | `switch(config)#` | 固定在 `management` VRF，配好 IP 才能 SSH/SNMP 远程管理 |
| 子接口 | `interface Ethernet1/1.100` + `encapsulation dot1q 100` | `switch(config-if)#` | 802.1Q 单臂路由；父接口需先 `no switchport` |
| 改 MTU | `mtu 9216` | `switch(config-if)#` | 见"陷阱"：9216 需配合全局 network-qos 才真生效 |
| 速率/双工 | `speed 10000` / `duplex full` | `switch(config-if)#` | 光口一般 `speed auto`；铜口对接异常时才手工钉死 |
| 开/关端口 | `no shutdown` / `shutdown` | `switch(config-if)#` | 注意 `system default switchport shutdown` 的全局影响 |
| 看接口总览 | `show interface status` | `switch#` | 一屏看 name/status/vlan/duplex/speed/type，日常最常用 |
| 看简要状态 | `show interface brief` | `switch#` | 含 mgmt0、port-channel、VLAN 接口 |
| 看三层状态 | `show ip interface brief` | `switch#` | 三层口与 SVI 的 admin/protocol 状态 |
| 看计数器 | `show interface Ethernet1/1 counters errors` | `switch#` | 排查 CRC/input error 丢包 |

## 典型示例

```text
switch# configure terminal
switch(config)# feature interface-vlan                        // 建 SVI 的前置
switch(config)# interface Ethernet1/1
switch(config-if)# description To-SPINE-01-Eth1/1
switch(config-if)# no switchport                             // 二层口切成三层路由口
switch(config-if)# ip address 10.1.12.1/30                    // 注意是斜线前缀，不是掩码
switch(config-if)# mtu 9216
switch(config-if)# no shutdown
switch(config-if)# exit
switch(config)# interface loopback0
switch(config-if)# ip address 10.255.0.1/32                   // router-id / 协议源地址
switch(config-if)# exit
switch(config)# interface vlan 10
switch(config-if)# description SERVER-VLAN-GW
switch(config-if)# ip address 10.10.10.1/24
switch(config-if)# no shutdown
switch(config-if)# exit
switch(config)# interface mgmt0
switch(config-if)# ip address 10.99.0.21/24                   // 带外管理，独立 management VRF
switch(config-if)# no shutdown
switch(config-if)# end
switch# show interface status
switch# show ip interface brief
switch# show interface Ethernet1/1 counters errors
switch# copy running-config startup-config
```

## 陷阱与版本差异

- **掩码写法**：NX-OS 只有 `ip address 10.1.1.1/24` 一种形式；从 IOS 复制过来的 `ip address 10.1.1.1 255.255.255.0` 会被拒。部分版本支持 `ip address 10.1.1.1/24 secondary`，多地址靠 `secondary` 关键字。
- **巨型帧要两步**：仅接口 `mtu 9216` 在 NX-OS 上不足以保证 9216 转发，还需全局 network-qos（`policy-map type network-qos` + `class type network-qos class-default` + `mtu 9216`，再 `system qos` 下 `service-policy type network-qos`），语法随平台版本有差异，以官方手册为准。
- 物理口在二层/三层间切换（`switchport` ↔ `no switchport`）会清掉该口原有二层或三层配置，并引起端口 flap。
- SVI 的 up 由 autostate 决定：VLAN 内无端口处于 STP forwarding 时 SVI 为 down，此时"IP 配了但不通"是正常现象；可用 `system default interface-vlan autostate` 相关命令调整（以官方手册为准）。
- `mgmt0` 属于 `management` VRF，不是 default VRF：跑 OSPF、写静态路由、`ping` 都要带 `vrf management`，否则不通。
- 接口计数器含义随平台/网络-qos 配置变化：官方计数器表说明 6400/10000 字节报文在不同 MTU 配置下的计数归类不同，别只看 "input error" 就判定链路故障。

## 高危操作 ⚠️

- ⚠️ 在承接管理的接口上执行 `shutdown` 或改 IP/掩码会导致远程失联，操作前确认 Console 或带外通道可用。
- ⚠️ 在跑 OSPF/BGP 的三层口上改 `ip address` 会立即拆除邻接并重收敛，需在维护窗口执行并备好旧地址。
- ⚠️ `no switchport` 会抹掉该口全部二层配置（VLAN、Trunk 允许列表），恢复时须整段重配，先 `show running-config interface Ethernet1/1` 留档。
- ⚠️ `default interface Ethernet1/1` 把端口参数一次性清成默认值，属于整段删除类操作，慎用。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Interfaces Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html)
2. [同书 · Configuring Layer 2 Interfaces（含 SVI autostate、计数器取值说明）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_0100.html)
3. [Cisco Nexus 9000 Series NX-OS Unicast Routing Configuration Guide, Release 10.3(x) · Configure static route（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_static_routing.html)
