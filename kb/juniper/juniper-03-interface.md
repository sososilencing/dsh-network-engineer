---
vendor: 瞻博（Juniper）
os: Junos
topic: 接口配置
devices: EX2300/EX4300 交换机、MX 系列路由器、SRX 系列防火墙
keywords: [接口, unit 0, family inet, address 斜线掩码, speed, link-mode, mtu, lo0, IRB, show interfaces terse]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/family-ethernet-switching-edit-interfaces-qfx-series-els.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/vlan-id-edit-vlans-qfx-series.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· 接口配置

> Junos 的接口是「物理口 › 逻辑单元 unit › family（协议族）› address」四层结构，IP 直接跟前缀长度写成 `10.0.0.1/24`，没有独立的子网掩码命令。同一物理口既能当二层口（`family ethernet-switching`，EX）也能当三层口（`family inet`），取决于 family 而不是 `switchport`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 描述 | `set interfaces ge-0/0/1 description "to-CORE-01"` | `[edit]` | 对应 Cisco `description` |
| 三层地址 | `set interfaces ge-0/0/1 unit 0 family inet address 10.1.1.1/24` | `[edit]` | 掩码写在斜线后，无 `netmask` 命令 |
| 第二地址 | `set interfaces ge-0/0/1 unit 0 family inet address 10.1.1.2/24` | `[edit]` | 同一 family 下可加多条 address |
| IPv6 地址 | `set interfaces ge-0/0/1 unit 0 family inet6 address 2001:db8::1/64` | `[edit]` | 需先启用 `family inet6` |
| Loopback | `set interfaces lo0 unit 0 family inet address 1.1.1.1/32` | `[edit]` | Router ID / 管理源地址常用 /32 |
| 速率 | `set interfaces ge-0/0/1 speed 1g` | `[edit]` | 取值随平台（1g/10g/auto 等，以官方手册为准） |
| 双工 | `set interfaces ge-0/0/1 link-mode full-duplex` | `[edit]` | 多数千兆及以上口固定全双工 |
| MTU | `set interfaces ge-0/0/1 mtu 9000` | `[edit]` | 物理口层级；改 MTU 需两端与路径一致 |
| 关闭口 | `set interfaces ge-0/0/1 disable` | `[edit]` | 对应 Cisco `shutdown`；恢复用 `delete ... disable` |
| 三层子接口（MX/SRX） | `set interfaces ge-0/1/0 vlan-tagging` + `set interfaces ge-0/1/0 unit 100 vlan-id 100 family inet address 10.100.0.1/24` | `[edit]` | 路由器侧 802.1Q 子接口，与 EX 二层口写法完全不同 |
| EX 上建三层 VLAN 口 | `set interfaces vlan unit 10 family inet address 10.0.0.1/24` | `[edit]` | 即 IRB/SVI，还需 `set vlans VLAN10 l3-interface vlan.10` |
| 查看接口简表 | `show interfaces terse` | 操作模式 | 一眼看 admin/link 状态与地址 |
| 查看详细 | `show interfaces ge-0/0/1 extensive` | 操作模式 | 含错误计数、协商结果、队列 |
| 查看光模块 | `show interfaces diagnostics optics ge-0/0/1` | 操作模式 | 收发光功率、温度、告警 |
| 查看统计 | `show interfaces statistics ge-0/0/1` | 操作模式 | 速率、包数、错包（光口排查必看） |
| 查看配置 | `show configuration interfaces ge-0/0/1 \| display set` | 操作模式 | 输出该口的 set 命令 |

## 典型示例

```text
user@EX-01> configure
[edit]
user@EX-01# set interfaces ge-0/0/1 description "to-CORE-01:ge-0/0/24"
user@EX-01# set interfaces ge-0/0/1 unit 0 family inet address 10.1.12.1/30   // 点对点互连，斜线掩码
user@EX-01# set interfaces ge-0/0/2 mtu 9000                                  // 存储网络大包
user@EX-01# set interfaces ge-0/0/3 disable                                    // 未使用口关闭
user@EX-01# set interfaces lo0 unit 0 family inet address 1.1.1.1/32           // Router ID
user@EX-01# set interfaces vlan unit 10 family inet address 10.0.0.1/24        // VLAN10 网关（IRB）
user@EX-01# set vlans VLAN10 l3-interface vlan.10
user@EX-01# show | compare
user@EX-01# commit confirmed 5
commit complete
user@EX-01# run show interfaces terse | match "ge-0/0/1|vlan"    // 核对 up/address
user@EX-01# run show interfaces ge-0/0/1 extensive | match error
user@EX-01# commit                                                // 确认
```

## 陷阱与版本差异

- **掩码不是独立命令**：`address 10.1.1.1/24` 里的 `/24` 就是掩码，写错前缀长度等于配错网段，且 commit 不会报错，只能用 `show interfaces terse` 复核。
- 物理口能不能配 IP，取决于有没有 `family inet`：EX 上若存在 `family ethernet-switching`，再加 `family inet` 会导致 commit 失败（一个单元只能属于一种转发模式）。
- **EX（交换）与 MX/SRX（路由/安全）写法不同**：EX 二层口 → `family ethernet-switching`；MX 三层子接口 → 先 `vlan-tagging` 再 `unit N vlan-id N family inet address ...`；SRX 三层口除配 IP 外还必须把接口加入 `security zones`，否则策略不生效。
- `unit` 号不是 VLAN 号：EX 上 `ge-0/0/1.0` 的 0 是逻辑单元，VLAN 归属靠 `vlan members`；MX 上 `unit 100` 才通常对应 VLAN 100。
- `disable` 与 `deactivate` 不同：`disable` 关的是接口本身（配置仍在），`deactivate` 是把整段语句挂起；排障时别混用。
- 速率/双工可选值随平台与光模块差异很大（如 `speed 1g` / `10g` / `auto`），不确定就用 `set interfaces ge-0/0/1 speed ?` 看当前平台补全（以官方手册为准）。
- 改 MTU 后 `show interfaces` 显示的是配置值，实际转发是否生效还要看对端与路径 MTU，IPv4 分片/PMTUD 问题常在此处暴露。

## 高危操作 ⚠️

- ⚠️ 修改管理口（`fxp0`/`me0`/`em0`）或 SSH 源地址所在接口的 IP：会立即断开远程会话，必须 `commit confirmed N`。
- ⚠️ 给承载业务的口加 `disable`：整段链路中断；批量关闭空闲口前先确认没有单臂路由/镜像/带外用途。
- ⚠️ 改 MTU：两端不一致会造成大包丢弃，表现为「ping 通、业务不通」，需在维护窗口两端同步。
- ⚠️ 删除 `lo0` 地址或改 Router ID 相关地址：OSPF/BGP 邻居会重建，路由震荡。

## 参考来源

1. [ethernet-switching（family）语句 · 逻辑单元与 family 关系（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/family-ethernet-switching-edit-interfaces-qfx-series-els.html)
2. [vlan-id · 三层子接口与 unit 编号规则（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/vlan-id-edit-vlans-qfx-series.html)
3. [CLI Configuration Mode Overview · 层级与语句路径（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html)
