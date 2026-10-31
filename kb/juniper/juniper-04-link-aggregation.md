---
vendor: 瞻博（Juniper）
os: Junos
topic: 链路聚合
devices: EX2300/EX4300/EX4600 交换机、QFX 系列、MX 系列路由器
keywords: [链路聚合, ae0, LAG, LACP, aggregated-ether-options, 802.3ad, device-count, aex, show lacp interfaces, link-protection]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/topics/topic-map/aggregated-ethernet-lacp-switches.html
  - https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/topics/topic-map/switches-interface-aggregated.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· 链路聚合

> Junos 的聚合口是 `ae`（aggregated Ethernet），必须先 `set chassis aggregated-devices ethernet device-count N` 声明数量，再把物理口挂进 `ae N`。LACP 默认 **passive**，与 Cisco/华为「默认不发 LACP」需显式 active 的直觉相似，但语句位置在 `aggregated-ether-options lacp`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 声明聚合口数量 | `set chassis aggregated-devices ethernet device-count 4` | `[edit]` | 不配则 `ae0` 无法 commit；改大需重新提交 |
| 建聚合口并启用 LACP | `set interfaces ae0 aggregated-ether-options lacp active` | `[edit]` | 对端可为 `passive`，两端都 passive 则聚合不 Up |
| LACP 报文周期 | `set interfaces ae0 aggregated-ether-options lacp periodic fast` | `[edit]` | `fast`=1s，`slow`=30s，缺省 1s |
| LACP 系统优先级 | `set interfaces ae0 aggregated-ether-options lacp system-priority 100` | `[edit]` | 数值小优先 |
| 挂成员口 | `set interfaces ge-0/0/1 ether-options 802.3ad ae0` | `[edit]` | 成员口只做这一步，不配 IP/不配 family |
| 二层 LAG（EX） | `set interfaces ae0 unit 0 family ethernet-switching interface-mode trunk vlan members [ VLAN10 VLAN20 ]` | `[edit]` | ELS 写法；老版本为 `port-mode` |
| 三层 LAG | `set interfaces ae0 unit 0 family inet address 10.0.0.1/30` | `[edit]` | MX 作三层聚合时常用 |
| 最小链路数 | `set interfaces ae0 aggregated-ether-options minimum-links 2` | `[edit]` | 缺省 1 条 Up 即整组 Up |
| 速率 | `set interfaces ae0 aggregated-ether-options link-speed 10g` | `[edit]` | 成员口速率需一致（部分平台支持混速） |
| 链路保护 | `set interfaces ae0 aggregated-ether-options link-protection` | `[edit]` | 支持 `non-revertive`，两端模式需一致 |
| 对端 LACP 能力弱时 | `set interfaces ae0 aggregated-ether-options lacp force-up` | `[edit]` | 对端不发 PDU 也想保持 Up 时用 |
| 查看 LACP | `show lacp interfaces` | 操作模式 | 看 actor/partner 状态（Current/Default） |
| 查看 LACP 统计 | `show lacp statistics interfaces ae0` | 操作模式 | 判断 PDU 是否互通 |
| 查看聚合口 | `show interfaces ae0 terse` | 操作模式 | 核对 up/down 与地址 |
| 查看成员 | `show interfaces ae0` | 操作模式 | 列出成员链路与聚合状态 |

## 典型示例

```text
user@EX-01> configure
[edit]
user@EX-01# set chassis aggregated-devices ethernet device-count 4    // 先声明，否则 ae0 无法提交
user@EX-01# set interfaces ae0 aggregated-ether-options lacp active  // 本端主动发 LACP
user@EX-01# set interfaces ae0 aggregated-ether-options lacp periodic fast
user@EX-01# set interfaces ae0 aggregated-ether-options minimum-links 2
user@EX-01# set interfaces ge-0/0/1 ether-options 802.3ad ae0        // 两个成员口
user@EX-01# set interfaces ge-0/0/2 ether-options 802.3ad ae0
user@EX-01# set interfaces ae0 unit 0 family ethernet-switching interface-mode trunk vlan members [ VLAN10 VLAN20 ]
user@EX-01# commit confirmed 5
commit complete
user@EX-01# run show lacp interfaces ae0      // 期望两个成员均为 Current / distributing
user@EX-01# run show lacp statistics interfaces ae0
user@EX-01# run show interfaces ae0 terse
user@EX-01# commit                              // 确认，取消定时回滚
```

## 陷阱与版本差异

- **漏配 `device-count`**：直接 `set interfaces ae0 ...` 会 commit 报错，这是新手最常见的坑；先声明数量再建口。
- LACP 默认 passive：两端都 passive 时 PDU 不交换、聚合不起来。跨厂商（对接 Cisco `channel-group mode active`、华为 `mode lacp-static`）时本端建议显式 `active`。
- 成员口进入 `ae` 后不要再给成员口配 `family inet` 或 `ethernet-switching`，否则 commit 失败；成员口只需 `ether-options 802.3ad aeN`。
- 单组 LAG 的成员上限与机型强相关：EX4200 为 8、EX4300 为 16、EX4600 为 32、QFX5100/EX4650 可达 64；超限 commit 报 group limit 错误（以官方手册为准）。
- ELS 与非 ELS 的二层成员写法不同（`interface-mode` / `port-mode`），EX 老版本升级后照抄新语法会报错。
- `link-protection` 时不要再手工把备份链路 `disable`，LACP 自行决定主备；且本地/对端 revertive 模式必须一致，否则主备状态反复切换。
- 对端是安全设备/服务器网卡绑定（mode 1/4 差异）时，LACP 可能不兼容，此时应改用不带 LACP 的静态 LAG 并确认对端支持。

## 高危操作 ⚠️

- ⚠️ 修改 `device-count` 后提交：超出当前数量的 `ae` 口会被删除，其上 VLAN/IP 一并丢失，上联可能瞬断。
- ⚠️ 一次性删除整组 `ae`（`delete interfaces ae0`）：成员口同时失去聚合配置，跨设备链路中断且失联风险高，必须 `commit confirmed N`。
- ⚠️ 调整 `minimum-links`：设大后成员口稍掉一个整组即 Down，可能造成比预期更严重的业务中断。
- ⚠️ 变更 LACP 模式（active↔passive）会触发重新协商，聚合口在数秒内不可用。

## 参考来源

1. [Aggregated Ethernet LACP for Switches（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/topics/topic-map/aggregated-ethernet-lacp-switches.html)
2. [Aggregated Ethernet Interfaces Overview · device-count / 成员上限（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/interfaces-ethernet-switches/topics/topic-map/switches-interface-aggregated.html)
3. [Commit the Configuration · 变更前比对与回滚（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html)
