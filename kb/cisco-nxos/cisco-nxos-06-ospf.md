---
vendor: 思科（Cisco）
os: NX-OS
topic: OSPF
devices: Nexus 3000/5000/7000/9000 系列（需 LAN_ENTERPRISE_SERVICES_PKG license）
keywords: [OSPF, feature ospf, router ospf, router-id, ip router ospf area, passive-interface, 邻居, show ip ospf neighbor, 重分发 route-map]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_ospfv2.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x.html
  - https://www.cisco.com/c/en/us/support/docs/switches/nexus-9000-series-switches/218333-understand-and-configure-nexus-9000-vpc.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · OSPF

> 适用于 Nexus 的 OSPFv2 单区域/多区域组网与 vPC 下的三层路由。
> **与 IOS 最关键的差异：NX-OS 不用 `network ... area` 宣告，而是在接口下 `ip router ospf 1 area 0`**。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 开 OSPF 特性 | `feature ospf` | `switch(config)#` | 硬前置，缺失时 `router ospf` 不可用；需 LAN_ENTERPRISE_SERVICES_PKG license |
| 起进程 | `router ospf 1` | `switch(config)#` | 进程号本地有效；支持多实例但同一接口不能属多实例 |
| 设 router-id | `router-id 10.255.0.1` | `switch(config-router)#` | 强烈建议手工指定（取 loopback0），否则选接口最大 IP，变更即断邻接 |
| 接口启用 OSPF | `ip router ospf 1 area 0` | `switch(config-if)#` | **核心差异**：在接口下宣告，不在进程下写 `network`；`0` 与 `0.0.0.0` 等价 |
| 设接口 cost | `ip ospf cost 100` | `switch(config-if)#` | 影响 SPF 计算；缺省按带宽参考值自动算 |
| 改网络类型 | `ip ospf network point-to-point` | `switch(config-if)#` | 与服务器/防火墙/点对点链路常用，可免 DR/BDR 选举 |
| 静默接口 | `passive-interface default` + `no passive-interface Ethernet1/1` | `switch(config-router)#` | 先全静默再放通上联，避免把接入口误宣告 |
| 区域类型 | `area 0.0.0.10 stub` / `area 0.0.0.10 nssa` | `switch(config-router)#` | 骨干 Area 0 不能配成 NSSA |
| 重分发 | `redistribute static route-map RM-STATIC` | `switch(config-router)#` | **必须**带 route-map，NX-OS 不支持裸 `redistribute` |
| 等价多路径 | `maximum-paths 8` | `switch(config-router)#` | 缺省值随版本，按官方手册核对 |
| 改 OSPF 管理距离 | `distance 110` | `switch(config-router)#` | 缺省 OSPF AD=110；调 AD 前先确认不与静态(1)/iBGP(200) 冲突 |
| RFC 兼容 | `rfc1583compatibility` | `switch(config-router)#` | 与只支持 RFC1583 的老设备互通时启用，默认按 RFC 2328 |
| 看邻居 | `show ip ospf neighbor` | `switch#` | 关注 State=FULL、Dead Time 是否递减 |
| 看接口参数 | `show ip ospf interface` | `switch#` | area、cost、hello/dead、网络类型、邻居数 |
| 看进程 | `show ip ospf` | `switch#` | 进程 router-id、SPF 次数、区域列表 |
| 看 OSPF 路由 | `show ip route ospf` | `switch#` | 路由表里标 `O`（域内）/`O IA`（域间）/`O E2` |
| 重启进程 | `restart ospf 1` | `switch#` | 平滑重启入口；官方说明 `restart ospf` 会触发 graceful restart |

## 典型示例

```text
switch# configure terminal
switch(config)# feature ospf                                     // 必做前置
switch(config)# interface loopback0
switch(config-if)# ip address 10.255.0.1/32
switch(config-if)# ip router ospf 1 area 0                       // loopback 也进 OSPF，作稳定 router-id
switch(config-if)# exit
switch(config)# interface Ethernet1/1
switch(config-if)# no switchport
switch(config-if)# ip address 10.1.12.1/30
switch(config-if)# ip router ospf 1 area 0                       // 接口级宣告，替代 IOS 的 network
switch(config-if)# ip ospf network point-to-point
switch(config-if)# no shutdown
switch(config-if)# exit
switch(config)# router ospf 1
switch(config-router)# router-id 10.255.0.1
switch(config-router)# passive-interface default                  // 先全静默
switch(config-router)# no passive-interface Ethernet1/1           // 只放通上联
switch(config-router)# maximum-paths 8
switch(config-router)# exit
switch# show ip ospf neighbor
switch# show ip ospf interface brief
switch# show ip route ospf
switch# copy running-config startup-config
```

## 陷阱与版本差异

- **没有 `network ... area` 命令**：NX-OS 的 OSPF 宣告只能在接口下 `ip router ospf 1 area <area-id>`。从 IOS 照抄 `network 10.1.12.0 0.0.0.3 area 0` 会被拒绝，是迁移期最常见的翻车点。
- `feature ospf` 未开时 `router ospf` 直接不可用；而 OSPF 属 LAN_ENTERPRISE_SERVICES_PKG license（官方 vPC 文档明确：OSPF/IS-IS 需该 license，LACP/HSRP/VRRP/vPC 在基础 license 内），license 不足时命令能配但不转发。
- **重分发必须配 route-map**：官方原文 "You must configure a route map with the redistribution to control which routes are passed into OSPFv2"，否则静态/直连路由不会真正进 OSPF。
- router-id 不手工指定时由系统自动选取，接口浮动/重启可能改变，导致邻接反复重建；多实例同 AS 时必须逐实例手工配 router-id（官方明确）。
- 与 IOS 的 RFC 行为差异：NX-OS 默认按 RFC 2328，IOS 侧若仍是 RFC1583 行为，外部路由选路会不一致，需在一侧加 `rfc1583compatibility` 对齐。
- 9.2(4) 起 NSSA ABR 对 P-bit 清零的默认路由 LSA 处理改为符合 RFC 3101，升级后可能"默认路由消失"，属行为变更而非故障。

## 高危操作 ⚠️

- ⚠️ `no feature ospf` 会删除全部 OSPF 配置（进程、接口 area、重分发），大网中等于一次性拆掉所有邻接；只想停协议请用 `router ospf 1` 下的 `shutdown`。
- ⚠️ 改 `router-id` 会重置该进程所有邻接（先 down 再重建），跨设备大网需逐台在窗口内执行。
- ⚠️ 在承载业务的三层口上误加 `passive-interface`（或用 `passive-interface default` 忘了 `no passive-interface` 放通上联）会让邻接静默消失，路由随之撤销，属典型的"改一行断一片"。
- ⚠️ `distance` 调低 OSPF 的 AD（如调到小于 1）会盖掉静态路由，反之调高会让浮动静态路由抢占转发，变更前先核对全网 AD 规划。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Unicast Routing Configuration Guide, Release 10.3(x) · OSPFv2（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_ospfv2.html)
2. [Cisco Nexus 9000 Series NX-OS Unicast Routing Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x.html)
3. [Understand and Configure Nexus 9000 vPC with Best Practices（license 与 `ip router ospf ... area` 实例，cisco.com）](https://www.cisco.com/c/en/us/support/docs/switches/nexus-9000-series-switches/218333-understand-and-configure-nexus-9000-vpc.html)
