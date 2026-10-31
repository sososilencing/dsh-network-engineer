---
vendor: Arista
os: EOS
topic: 链路聚合 Port-Channel / LACP
devices: vEOS、DCS-7050X/7060X/7280R/7500R、CCS-720XP 等 EOS 设备
keywords: [Port-Channel, channel-group, LACP, mode active, mode passive, mode on, lacp fallback, 负载均衡哈希, MLAG, show port-channel]
sources:
  - https://www.arista.com/en/um-eos/eos-port-channels-and-lacp
  - https://www.arista.com/en/um-eos/eos-layer-2-configuration
  - https://www.arista.com/en/um-eos/eos-virtual-lans-vlans
verified: high
updated: 2026-10-30
---

# Arista · 链路聚合 Port-Channel / LACP

> 适用于 EOS 的 Port-Channel 聚合：LACP 动态协商（`mode active`/`passive`）与静态（`mode on`）。VLAN 与三层 IP 一律配在 `Port-Channel` 上，成员口只负责加入。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建聚合口 | `interface Port-Channel1` | `(config)#` | 逻辑口，编号 1–999999 随平台 |
| 成员加入（LACP 主动） | `channel-group 1 mode active` | 成员接口视图 | 互联推荐；两端都发 LACP |
| 成员加入（LACP 被动） | `channel-group 1 mode passive` | 成员接口视图 | 只应答不主动，至少一端为 active |
| 成员加入（静态） | `channel-group 1 mode on` | 成员接口视图 | 不协商，**对端也必须 on** |
| 批量加入成员 | `interface Ethernet 1-2` → `channel-group 1 mode active` | 成员接口视图 | 一次进多个口再统一加入 |
| LACP 回退（静态） | `port-channel lacp fallback static` | 聚合口视图 | 对端未起 LACP 时保持一个成员可用 |
| LACP 回退（单口） | `port-channel lacp fallback individual` | 聚合口视图 | 回退后成员以独立口转发 |
| 回退超时 | `port-channel lacp fallback timeout 30` | 聚合口视图 | 秒；参数随版本，以官方手册为准 |
| 负载均衡算法 | `port-channel load-balance src-dst-ip` | `(config)#` | 基于 L3/L4 哈希，默认随平台 |
| 聚合口配 Trunk | `switchport mode trunk` → `switchport trunk allowed vlan 10,20,30` | 聚合口视图 | **不要**在成员口单独配 VLAN |
| 聚合口做三层 | `no switchport` → `ip address 10.0.0.1/30` | 聚合口视图 | 需全局 `ip routing` |
| 查看聚合摘要 | `show port-channel` | `>` 特权 EXEC | 各 Port-Channel 成员与状态 |
| 查看聚合详情 | `show port-channel 1 detailed` | `>` 特权 EXEC | 哈希算法、成员活动状态 |
| 查看 LACP 邻居 | `show lacp neighbor` | `>` 特权 EXEC | 对端 system-id 与端口状态 |
| 查看聚合口统计 | `show interfaces Port-Channel1` | `>` 特权 EXEC | 流量与错误计数 |
| 跨设备聚合 | `mlag configuration` | `(config)#` | MLAG：两台交换机做跨设备 LAG，配置与一致性要求另见官方指南 |

## 典型示例

```text
switch# configure terminal
switch(config)# port-channel load-balance src-dst-ip     // 按源/目的 IP 哈希
switch(config)# interface Ethernet 1-2
switch(config-if-Et1-2)# channel-group 1 mode active      // LACP 主动模式
switch(config-if-Et1-2)# exit
switch(config)# interface Port-Channel1
switch(config-if-Po1)# switchport mode trunk
switch(config-if-Po1)# switchport trunk allowed vlan 10,20,30
switch(config-if-Po1)# port-channel lacp fallback static  // 可选：对端未起 LACP 时保底
switch(config-if-Po1)# exit
switch(config)# end
switch# copy running-config startup-config                // ★ 不保存则重启丢失
switch# show port-channel
switch# show lacp neighbor
switch# show interfaces Port-Channel1
```

三层聚合口写法：

```text
switch(config)# ip routing
switch(config)# interface Port-Channel10
switch(config-if-Po10)# no switchport
switch(config-if-Po10)# ip address 10.0.0.1/30
switch(config-if-Po10)# exit
switch(config)# interface Ethernet 3-4
switch(config-if-Et3-4)# channel-group 10 mode active
switch(config-if-Et3-4)# exit
```

## 陷阱与版本差异

- 两端模式必须匹配：一端 `mode active` 一端 `mode on`，LACP 对静态端无感，结果是一个成员被挂起、带宽减半或整条不通。
- `mode active` + `mode passive` 可以配对（只要有一端 active）；`mode on` 必须两端都是 `on`。
- 成员口的速率、双工、MTU、`switchport mode` 必须一致；EOS 会把不一致的成员置为不可用，表现为 `show port-channel` 里成员 up 但不在活动列表。
- VLAN / IP **只配在 Port-Channel 上**；成员口上残留的 `switchport access vlan` 会造成流量走错广播域。
- 哈希算法改的是全局 `port-channel load-balance`，对大流量场景影响明显；实际效果取决于流数量，单流不会分摊。
- MLAG 是**跨设备**聚合，涉及 peer-link、peer-address、`dual-active` 检测与两端配置一致性，与单机 Port-Channel 完全不同，请按官方 MLAG 指南单独实施。

## 高危操作 ⚠️

- ⚠️ 成员口加入/退出聚合、切换 `mode` 会造成该链路**短暂中断**，业务口须在维护窗口操作。
- ⚠️ `no interface Port-Channel1` 删除聚合口会连带删除其上所有 VLAN/IP 配置，且成员口回落到独立二层口，可能立即形成环路或黑洞。
- ⚠️ 改全局 `port-channel load-balance` 会让现网所有聚合口的流量重新哈希，短时出现乱序/丢包，避开业务高峰。
- ⚠️ MLAG 场景下只改单侧 `mlag` 参数或 `peer-link`，可能触发双主（dual-active），整对交换机转发异常。

## 参考来源

1. [EOS User Manual · Port Channels and LACP（arista.com）](https://www.arista.com/en/um-eos/eos-port-channels-and-lacp)
2. [EOS User Manual · Layer 2 Configuration（arista.com）](https://www.arista.com/en/um-eos/eos-layer-2-configuration)
3. [EOS User Manual · Virtual LANs (VLANs)（arista.com）](https://www.arista.com/en/um-eos/eos-virtual-lans-vlans)
