---
vendor: 思科（Cisco）
os: NX-OS
topic: VLAN 与二层
devices: Nexus 3000/5000/7000/9000 系列（Nexus 9300 默认二层，9300-EX 默认三层）
keywords: [VLAN, switchport mode trunk, switchport access vlan, trunk allowed vlan, native vlan, feature interface-vlan, show vlan brief, spanning-tree mode, 二层端口]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_0100.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_static_routing.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · VLAN 与二层

> 适用于 Nexus 的 VLAN 创建、Access/Trunk 端口、允许 VLAN 列表、Native VLAN 与 STP 模式。
> 与 IOS 的差别：端口默认三层还是二层**随平台型号不同**，且建 SVI 前必须先 `feature interface-vlan`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建 VLAN | `vlan 10` | `switch(config)#` | 进 VLAN 配置模式；范围 1–4094，另有内部保留 VLAN |
| VLAN 命名 | `name WEB-SERVER` | `switch(config-vlan)#` | 便于审计，`show vlan` 可见 |
| 建 SVI 前置 | `feature interface-vlan` | `switch(config)#` | **不开则无法建 VLAN 接口**，官方明确要求 |
| 端口切二层 | `switchport` | `switch(config-if)#` | 9300-EX 等默认三层端口，必须先切回二层 |
| 改全局默认二层 | `system default switchport` | `switch(config)#` | 让新端口默认二层，等同 IOS 习惯 |
| 配 Access 口 | `switchport mode access` + `switchport access vlan 10` | `switch(config-if)#` | Access VLAN 必须先创建，否则端口会被 shut down |
| 配 Trunk 口 | `switchport mode trunk` | `switch(config-if)#` | 只支持 802.1Q 封装，无 `switchport trunk encapsulation` 概念 |
| 允许 VLAN 列表 | `switchport trunk allowed vlan 10,20,30` | `switch(config-if)#` | 也可 `add`/`remove`；缺省全部 VLAN 都允许 |
| 原生 VLAN | `switchport trunk native vlan 99` | `switch(config-if)#` | 两端必须一致；缺省为 VLAN 1 |
| 关闭端口 | `shutdown` | `switch(config-if)#` | 注意 `system default switchport shutdown` 影响新端口 |
| 看全部 VLAN | `show vlan brief` | `switch#` | 等价 IOS 的 `show vlan brief` |
| 看单个 VLAN | `show vlan id 10` | `switch#` | 含端口成员列表 |
| 看端口二层属性 | `show interface Ethernet1/1 switchport` | `switch#` | 管理/路由/Trunk 状态一目了然 |
| 看 Trunk 协商 | `show interface Ethernet1/1 trunk` | `switch#` | 实际放通的 VLAN 列表 |
| 看 STP | `show spanning-tree` / `show spanning-tree summary` | `switch#` | 输出显示 `protocol rstp` 即缺省快速生成树 |
| 改 STP 模式 | `spanning-tree mode mst` | `switch(config)#` | 缺省 `rapid-pvst`；MST 需显式切换并配 `spanning-tree mst configuration` |
| 看 MAC 表 | `show mac address-table` | `switch#` | 支持 `| include`、`vlan 10` 过滤 |

## 典型示例

```text
switch# configure terminal
switch(config)# feature interface-vlan                  // 起 SVI 的硬前置
switch(config)# vlan 10
switch(config-vlan)# name WEB-SERVER
switch(config-vlan)# vlan 20
switch(config-vlan)# name DB-SERVER
switch(config-vlan)# exit
switch(config)# interface Ethernet1/10
switch(config-if)# switchport                         // 平台默认三层时必做，切回二层
switch(config-if)# switchport mode access             // 接服务器/终端的接入口
switch(config-if)# switchport access vlan 10
switch(config-if)# no shutdown
switch(config-if)# exit
switch(config)# interface Ethernet1/48
switch(config-if)# switchport
switch(config-if)# switchport mode trunk               // 上行到核心/汇聚的干道口
switch(config-if)# switchport trunk allowed vlan 10,20,99
switch(config-if)# switchport trunk native vlan 99     // 两端必须一致
switch(config-if)# no shutdown
switch(config-if)# end
switch# show vlan brief
switch# show interface Ethernet1/48 switchport
switch# show interface Ethernet1/48 trunk
switch# show spanning-tree summary
switch# copy running-config startup-config
```

## 陷阱与版本差异

- **端口缺省二层/三层随平台型号而变**：官方说明 Nexus 9300 全部端口默认 Layer 2，而 Nexus 9300-EX 默认 Layer 3。跨型号组网时先用 `show interface switchport` 确认，再决定是否下发 `switchport`。
- Access VLAN 指向一个**尚未创建**的 VLAN 时，系统会把该 Access 端口 shut down（官方明确），表现为"配完不通"，务必先建 VLAN。
- 二层口改三层（`no switchport`）会**清除该口全部二层配置**并恢复默认 VLAN 设置；反之亦然，来回切换等于重配。
- Trunk 缺省允许所有 VLAN；生产建议显式 `switchport trunk allowed vlan`，并考虑把 VLAN 1 从允许列表移除以缩小 STP 域（官方建议）。
- Native VLAN 不匹配会造成 VLAN 1/原生 VLAN 流量单向丢弃 + CDP/LLDP 邻接异常；FCoE VLAN 不能做以太 Trunk 的 native VLAN。
- STP 缺省模式为 `rapid-pvst`（`show spanning-tree` 输出中的 `protocol rstp` 即此模式），与 IOS 的 `rapid-pvst` 同名但命令是 `spanning-tree mode rapid-pvst`/`mst`；MST 需另配 `spanning-tree mst configuration` 且域名/修订号两端一致。
- SVI 的 up/down 由 autostate 决定：该 VLAN 至少要有一个口处于 STP forwarding，SVI 才 up；只有 `no shutdown` 而 VLAN 无成员口时 SVI 仍 down。

## 高危操作 ⚠️

- ⚠️ `no vlan 10` 会删除 VLAN 及其 SVI，所有挂在该 VLAN 的业务立即中断；先 `show vlan id 10` 盘点成员端口，并准备回退脚本。
- ⚠️ 在承载管理的 Trunk 上改 `switchport trunk allowed vlan`（尤其用 `remove` 误删管理 VLAN）会导致远程失联，务必保留带外管理通道。
- ⚠️ 改 `switchport trunk native vlan` 会造成瞬时环路风险与 STP 重收敛，需在维护窗口做，并两端同时改。
- ⚠️ `system default switchport shutdown` 会让此后新接入的端口默认关闭，批量上线前确认该全局默认值。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Interfaces Configuration Guide, Release 10.3(x) · Configuring Layer 2 Interfaces（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_0100.html)
2. [Cisco Nexus 9000 Series NX-OS Interfaces Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html)
3. [Cisco Nexus 9000 Series NX-OS Unicast Routing Configuration Guide, Release 10.3(x) · Configure static route（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_static_routing.html)
