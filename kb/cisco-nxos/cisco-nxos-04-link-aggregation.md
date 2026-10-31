---
vendor: 思科（Cisco）
os: NX-OS
topic: 链路聚合与 vPC
devices: Nexus 3000/5000/7000/9000 系列（跨设备聚合需 vPC 支持的平台与 license）
keywords: [port-channel, channel-group mode active, feature lacp, LACP, 手工聚合, 负载分担, vPC, vpc domain, peer-link, peer-gateway, show port-channel summary]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_010000.html
  - https://www.cisco.com/c/en/us/support/docs/switches/nexus-9000-series-switches/218333-understand-and-configure-nexus-9000-vpc.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · 链路聚合与 vPC

> 适用于 Nexus 的 port-channel（手工 `on` / LACP `active`）与跨设备 vPC 双活接入。
> 与 IOS 差异：**LACP 需先 `feature lacp`**；**不支持 PAgP**；跨框聚合必须用 vPC，不是堆叠。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 开 LACP 特性 | `feature lacp` | `switch(config)#` | 不下发则 `mode active` 不可用；跨框 vPC 也依赖它 |
| 建聚合口 | `interface port-channel 1` | `switch(config)#` | 先建逻辑口或由成员口自动创建都可以 |
| LACP 主动模式 | `channel-group 1 mode active` | `switch(config-if)#` | 互联首选，能协商并检测成员链路 |
| LACP 被动模式 | `channel-group 1 mode passive` | `switch(config-if)#` | 等对端发起，对端必须 `active` |
| 手工聚合 | `channel-group 1 mode on` | `switch(config-if)#` | 不跑协议，两端都须 `on`；对端不支持 LACP 时用 |
| 成员口批量加入 | `interface Ethernet1/1-4` 后逐个 `channel-group 1 mode active` | `switch(config)#` | NX-OS 无 `channel-group ... to` 批量语法 |
| 最少活动链路 | `lacp min-links 2` | `switch(config-if)#` | LACP 口成员低于阈值整条 down（参数随版本，以官方手册为准） |
| 最大成员数 | `lacp max-bundle 8` | `switch(config-if)#` | 限制参与转发的成员口数量 |
| 聚合口三层化 | `no switchport` + `ip address 10.1.1.1/24` | `switch(config-if)#` | 三层地址一律配在 port-channel 上，成员口不配 |
| 全局负载算法 | `port-channel load-balance src-dst ip-l4port` | `switch(config)#` | **全设备唯一**，不能按单个 port-channel 设 |
| 看聚合汇总 | `show port-channel summary` | `switch#` | 看成员口状态（P=up in bundle、I=individual、S=suspended） |
| 看兼容性检查项 | `show port-channel compatibility-parameters` | `switch#` | 成员口加不进去时查这张全量检查清单 |
| 看负载算法 | `show port-channel load-balancing` | `switch#` | 核对 L2/L3/组播各自使用的哈希字段 |
| 建 vPC 域 ⚠️license | `feature vpc` + `vpc domain 1` | `switch(config)#` | vPC 在 NX-OS 基础 license 内；OSPF/IS-IS 需 LAN_ENTERPRISE_SERVICES_PKG |
| 指定 peer-keepalive | `peer-keepalive destination 10.201.182.26 source 10.201.182.25` | `switch(config-vpc-domain)#` | 建议走 mgmt0/独立链路，不走 peer-link |
| 网关双活 | `peer-gateway` | `switch(config-vpc-domain)#` | 允许对端 MAC 也能本地转发，避免绕 peer-link；须两端都配 |
| STP 双活根 | `peer-switch` | `switch(config-vpc-domain)#` | 让两台成为同一 STP 根，两端都必须配才生效 |
| 指定 peer-link | `interface port-channel10` + `vpc peer-link` | `switch(config-if)#` | peer-link 建议 10G 以上多链路聚合 |
| 声明成员 vPC | `interface port-channel20` + `vpc 20` | `switch(config-if)#` | vPC 号全局一致，两端都要配 |
| 看 vPC 状态 | `show vpc brief` / `show vpc role` | `switch#` | 关注 peer adjacency、keep-alive、consistency 三项 |

## 典型示例

```text
switch# configure terminal
switch(config)# feature lacp                       // LACP 前置开关
switch(config)# interface port-channel 1
switch(config-if)# switchport mode trunk
switch(config-if)# switchport trunk allowed vlan 10,20
switch(config-if)# no shutdown
switch(config-if)# exit
switch(config)# interface Ethernet1/1-4            // NX-OS 支持区间写法
switch(config-if-range)# channel-group 1 mode active
switch(config-if-range)# no shutdown
switch(config-if-range)# exit
switch# show port-channel summary
switch# copy running-config startup-config
```

```text
// vPC（跨设备聚合）骨架：两台 Nexus 逻辑上当成一台，接服务器/下行交换机做双活
switch(config)# feature vpc
switch(config)# vpc domain 1
switch(config-vpc-domain)# peer-switch                                  // 两台同一 STP 根
switch(config-vpc-domain)# peer-keepalive destination 10.201.182.26 source 10.201.182.25
switch(config-vpc-domain)# peer-gateway                                // 网关双活，少绕 peer-link
switch(config-vpc-domain)# ip arp synchronize
switch(config-vpc-domain)# exit
switch(config)# interface port-channel 10
switch(config-if)# vpc peer-link                                       // 专用 peer-link
switch(config-if)# exit
switch(config)# interface port-channel 20
switch(config-if)# vpc 20                                              // 面向下行的 vPC
switch(config-if)# exit
switch# show vpc brief
switch# show vpc role
switch# show vpc consistency-parameters global                         // 一致性检查
```

## 陷阱与版本差异

- **不支持 PAgP**（官方明确）：从 IOS 迁移过来的 `channel-group 1 mode desirable/auto` 在 NX-OS 直接报错，只能改 `active/passive/on`。
- 单条 port-channel 最多 32 个活动成员；成员口必须同速率、同双工、同二层/三层属性，否则被挂起（suspended）。加不进去先跑 `show port-channel compatibility-parameters` 对齐检查项。
- 成员口一旦入聚合，**switchport/三层配置必须配在 port-channel 上**，成员口再配无效；聚合口删除时系统会把成员口视为已移出。
- 负载算法是全设备唯一（`port-channel load-balance`），不能按端口组区分；缺省 L2 为 `src-dst-mac`、L3 为 `src-dst ip-l4port`。多层级联时不变算法会出现极化（polarization）。
- vPC 与 ISSU：vPC 环境下升级应逐台 `install all nxos bootflash:<image>`，一台升级时另一台的配置会被自动锁定，别两台同时升级。
- vPC 平台/license 限制：vPC 功能本身在基础 NX-OS license 内，但 `layer3 peer-router`、VXLAN 等按平台与版本支持情况不同；启用 `peer-gateway` 必须在 `layer3 peer-router` 之前，且两端都要配才生效（以官方手册为准）。

## 高危操作 ⚠️

- ⚠️ `no interface port-channel 1` 会拆掉整条聚合，所有成员链路业务同时中断并触发 STP 重收敛；先 `show port-channel summary` 留档，安排窗口。
- ⚠️ 成员口 `channel-group` 变更（切 mode、移出/加入）会造成该链路 flap，跨设备 vPC 场景下可能引发 vPC 一致性告警。
- ⚠️ vPC peer-link 中断会触发"双主"风险：务必配 `auto-recovery`（可设 `reload-delay`）并保证 keep-alive 链路与 peer-link 物理分离，否则恢复 peer-link 时可能出现短暂环路。
- ⚠️ 在已带业务的两台设备上首次建 vPC，会重建 STP 拓扑并清空原有阻塞端口状态；上线前完成 VLAN/MTU/STP 参数的 `show vpc consistency-parameters` 预检。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Interfaces Configuration Guide, Release 10.3(x) · Configuring Port Channels（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_010000.html)
2. [Understand and Configure Nexus 9000 vPC with Best Practices（Document ID 218333，cisco.com）](https://www.cisco.com/c/en/us/support/docs/switches/nexus-9000-series-switches/218333-understand-and-configure-nexus-9000-vpc.html)
3. [Cisco Nexus 9000 Series NX-OS Interfaces Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html)
