---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: 链路聚合
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [EtherChannel, channel-group, LACP active, PAgP desirable, on 模式, Port-channel, lacp max-bundle, port-channel load-balance, show etherchannel summary]
sources:
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/lyr2/b_176_lyr2_9300_cg/configuring_etherchannels.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/lyr2/b_176_lyr2_9300_cg.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlan_trunks.html
verified: high
updated: 2026-10-30
---

# 思科 · 链路聚合（EtherChannel）

> 适用于 IOS/IOS-XE 设备间的链路捆绑：三层互通、堆叠上联与服务器双网卡。思科同时支持 IEEE 802.3ad LACP（`active`/`passive`）、私有 PAgP（`desirable`/`auto`）与不协商的 `on` 模式；跨厂商对接一律用 LACP。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建聚合逻辑口 | `interface port-channel 1` | `(config)#` | 编号 1–128，与 channel-group 号对应 |
| 成员口加入（LACP） | `channel-group 1 mode active` | `(config-if)#` | 主动发 LACP 报文，跨厂商首选 |
| LACP 被动模式 | `channel-group 1 mode passive` | `(config-if)#` | 只响应；两端都 passive 建不起来 |
| PAgP 主动模式 | `channel-group 1 mode desirable` | `(config-if)#` | 思科私有，只能与思科设备对接 |
| 手工模式 | `channel-group 1 mode on` | `(config-if)#` | 不协商；两端必须都 `on`，最易出环 |
| 三层聚合口 | `no switchport`（在 port-channel 下） | `(config-if)#` | L3 EtherChannel 需手工建逻辑口并转路由口 |
| 聚合口上配 Trunk | `switchport trunk allowed vlan 10,20` | `(config-if)#` | 参数只需配在 port-channel 上，自动同步成员口 |
| LACP 端口优先级 | `lacp port-priority 100` | `(config-if)#` | 数值小者优先，用于 1:1 主备 |
| LACP 系统优先级 | `lacp system-priority 100` | `(config)#` | 影响对端选主，两端建议错开 |
| 最大成员数 | `lacp max-bundle 4` | `(config-if)#` | 超出成员自动成为 hot-standby |
| 最小链路数 | `port-channel min-links 2` | `(config-if)#` | 成员不足时聚合口 down，防带宽不足 |
| 负载分担算法 | `port-channel load-balance src-dst-ip` | `(config)#` | 全局对所有聚合口生效 |
| 查看聚合摘要 | `show etherchannel summary` | 特权 EXEC `#` | 关注 `(SU)`＝up 可用、`(SD)`＝down |
| 查看成员与协商 | `show etherchannel port-channel` | 特权 EXEC `#` | 每成员口的 flags 与端口状态 |
| 查看 LACP 邻居 | `show lacp neighbor` | 特权 EXEC `#` | 确认对端 system-id 与端口状态 |

## 典型示例

```text
SW1# configure terminal
SW1(config)# interface port-channel 10
SW1(config-if)# switchport mode trunk                  // L2 聚合：参数配在逻辑口
SW1(config-if)# switchport trunk allowed vlan 10,20
SW1(config-if)# interface range gigabitethernet 1/0/1 - 2
SW1(config-if-range)# switchport mode trunk
SW1(config-if-range)# channel-group 10 mode active      // LACP 主动，对端 active/passive 均可
SW1(config-if-range)# end
SW1# show etherchannel summary
SW1# show lacp neighbor
SW1# copy running-config startup-config
```

三层聚合口（对接路由器/防火墙）：

```text
SW1# configure terminal
SW1(config)# interface port-channel 20
SW1(config-if)# no switchport                          // 先转路由口，再绑成员
SW1(config-if)# ip address 10.1.20.1 255.255.255.0
SW1(config-if)# interface range gigabitethernet 1/0/3 - 4
SW1(config-if-range)# no switchport
SW1(config-if-range)# channel-group 20 mode active
SW1(config-if-range)# end
SW1# show etherchannel port-channel
SW1# show ip interface brief
```

## 陷阱与版本差异

- 成员口必须"兼容"：同一 EtherChannel 内端口要在同一 VLAN 或都配成 trunk（官方 Restrictions），速率/双工/VLAN 不一致的成员会被 suspend，现象是聚合口 up 但只有部分链路转发。
- `on` 模式不做任何协商：一端 `on`、另一端 `active` 时对端不发/不认 LACP，会出现成员口反复 up/down 甚至 STP 环路；官方对 `on` 模式专门给出了 caution 提示。
- LACP 的 `passive`+`passive` 组合永远协商不起来（都等对方发起）；PAgP 的 `auto`+`auto` 同理。
- 每个 EtherChannel 最多 8 个兼容端口（官方概述）；三层聚合必须先 `interface port-channel` + `no switchport` 再配 `channel-group`，顺序反了成员口进不了 bundle。
- 成员口一旦入组，二层/三层参数变更应在 `port-channel` 上做；直接改成员口会导致该口被移出 bundle（官方 VLAN 章节 restrictions）。

## 高危操作 ⚠️

- ⚠️ 错用 `on` 模式（两端配置不一致）：可能出现丢包与二层环路，业务大范围受影响；变更前确认两端模式匹配，优先使用 LACP `active`。
- ⚠️ `no interface port-channel 10`：删除逻辑口会同时移除成员口的 channel-group 绑定，链路瞬间中断且需重建；操作前备份配置并准备维护窗口。
- ⚠️ 对成员口做 `shutdown`、换模块或改速率：触发重新哈希与短时丢包，实时业务（语音/视频）会感知；尽量在低峰执行并逐口操作。
- ⚠️ `port-channel load-balance` 全局调整算法会重算所有聚合口的哈希分布，可能造成瞬时乱序与丢包；变更后验证关键业务流。

## 参考来源

1. [Configuring EtherChannels（模式表、Restrictions、监控命令）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/lyr2/b_176_lyr2_9300_cg/configuring_etherchannels.html)
2. [Layer 2 Configuration Guide, Cisco IOS XE 17.6.x (Catalyst 9300)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/lyr2/b_176_lyr2_9300_cg.html)
3. [Configuring VLAN Trunks（聚合口与 trunk 参数一致性要求）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/vlan/b_176_vlan_9300_cg/configuring_vlan_trunks.html)
