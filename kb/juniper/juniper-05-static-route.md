---
vendor: 瞻博（Juniper）
os: Junos
topic: 静态路由
devices: EX2300/EX4300 三层交换机、MX 系列路由器、SRX 系列防火墙
keywords: [静态路由, routing-options, static route, next-hop, preference, 浮动静态, discard, reject, qualified-next-hop, show route]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config_static-routes.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/static-edit-routing-options.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· 静态路由

> 静态路由统一写在 `[edit routing-options]` 下，语法是 `static route 目的/前缀 next-hop 下一跳`。Junos 的 `preference` **数值越小越优先**：直连 0、静态 5、OSPF 内部 10、RIP 100、OSPF 外部（AS External）150、BGP 170。做浮动静态就是把备份路由的 preference 调大，而不是像 Cisco 那样加 AD 值。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 普通静态路由 | `set routing-options static route 192.168.47.0/24 next-hop 172.16.1.2` | `[edit]` | 下一跳不可达时路由不装入转发表 |
| 默认路由 | `set routing-options static route 0.0.0.0/0 next-hop 10.1.12.1` | `[edit]` | 也可写 `default`，等价 0.0.0.0/0 |
| IPv6 默认路由 | `set routing-options static route ::/0 next-hop 2001:db8:1::1` | `[edit]` | 同一 `static route` 语句下按前缀区分 v4/v6 |
| 浮动静态（备份） | `set routing-options static route 192.168.47.0/24 next-hop 172.16.1.6 preference 200` | `[edit]` | preference 大于主路由（如 OSPF 10、静态 5）即成为备份 |
| 主路由显式降优先 | `... next-hop 172.16.1.2 preference 5` | `[edit]` | 缺省静态 preference 即 5，可显式写明便于审计 |
| 黑洞路由（静默丢弃） | `set routing-options static route 10.99.0.0/16 discard` | `[edit]` | 丢包且不回 ICMP，源端表现为超时 |
| 拒绝路由（回 ICMP） | `set routing-options static route 10.99.0.0/16 reject` | `[edit]` | 丢包并回不可达，便于快速定位 |
| 多下一跳负载分担 | `set routing-options static route 0.0.0.0/0 next-hop [ 10.1.12.1 10.1.13.1 ]` | `[edit]` | 多条等价路径共同装表 |
| 按下一跳设优先级 | `set routing-options static route 0.0.0.0/0 qualified-next-hop 10.1.13.1 preference 7` | `[edit]` | 主备同写一条路由，按 qualified-next-hop 的 preference 选路 |
| 打标签 | `... next-hop 172.16.1.2 tag 100` | `[edit]` | 便于路由策略匹配与重分发 |
| 不解析下一跳 | `... next-hop 192.0.2.1 no-resolve` | `[edit]` | 下一跳不在直连网段时避免解析问题 |
| 查看路由表 | `show route` | 操作模式 | `[Protocol/Preference]` 方括号内即为协议与优先级 |
| 只看静态 | `show route protocol static` | 操作模式 | 核对是否 Active（`*` 标记） |
| 看明细 | `show route 0.0.0.0/0 detail` | 操作模式 | 含 preference、metric、下一跳与年龄 |
| 看转发表 | `show route forwarding-table destination 192.168.47.1` | 操作模式 | 确认真正下发到转发面 |

## 典型示例

```text
user@R1> configure
[edit]
user@R1# set routing-options static route 192.168.47.0/24 next-hop 172.16.1.2      // 主路由，preference 缺省 5
user@R1# set routing-options static route 192.168.47.0/24 next-hop 172.16.2.2 preference 200   // 备份：preference 更大
user@R1# set routing-options static route 0.0.0.0/0 next-hop 10.1.12.1             // 默认路由
user@R1# set routing-options static route 10.99.0.0/16 discard                     // 黑洞汇总
user@R1# set routing-options static route 172.16.99.0/24 reject                    // 明确回不可达
user@R1# commit confirmed 5
commit complete
user@R1# run show route 192.168.47.0/24 detail      // 看 [Static/5] 与备份 [Static/200]
user@R1# run show route protocol static
user@R1# run show route forwarding-table destination 192.168.47.1
user@R1# commit                                       // 确认
```

## 陷阱与版本差异

- **preference 方向与 Cisco AD 相反**：Junos 数值小者优先（静态 5 优于 OSPF 内部 10）；Cisco 是数值小者优先但静态为 1。把 Cisco 的「AD 250 做浮动」直译成 Junos 时必须写 `preference 250`（大于 5/10 即可），不要写 `preference 1`。
- 静态路由只有在**下一跳可达**时才进入路由表并下发转发表；下一跳所在接口 Down 时该路由自动撤销，这是「配了却不通」的第一排查点。
- `discard` 与 `reject` 差别很大：`discard` 静默丢弃（源端只看到超时），`reject` 回 ICMP 不可达；排障期用 `reject`、防环/防攻击用 `discard`。
- 同一条路由配两条 `next-hop` 是等价多路径负载分担；要做主备必须用两个 `preference` 或 `qualified-next-hop` 的 `preference`，别指望顺序生效。
- `0.0.0.0/32` 不是默认路由：Junos 缺省丢弃目的 0.0.0.0 的包，需要时才显式加 `0.0.0.0/32` 的静态路由（IPv6 为 `0::0/128`）——这是官方文档明确点出的特例。
- 管理/内部以太接口（fxo 等）不支持多下一跳负载分担；在 fxp 接口上无法同时配多下一跳（以官方手册为准）。
- 部分平台在 QFX5xxx/EX4xxx 上对 `next-table` 类静态路由、超短前缀（IPv4 < /16）会走软件转发，性能下降，规划时避免。

## 高危操作 ⚠️

- ⚠️ 删除或改动默认路由（`delete routing-options static route 0.0.0.0/0`）：远程管理流量若走默认路由，会立即失联，必须 `commit confirmed N`。
- ⚠️ 直接改主路由 preference 而不先确认备份路径可用：切换瞬间黑洞，业务中断。
- ⚠️ 大面积 `discard` 汇总：会把业务网段一并吞掉且无 ICMP 提示，故障定位会被误导。
- ⚠️ 多人同时改 `routing-options`：`commit` 是全量提交，可能连带提交他人未完成的静态路由改动，建议 `configure exclusive`。

## 参考来源

1. [Configure Static Routes · 静态路由 preference 5（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/static-routing/topics/topic-map/config_static-routes.html)
2. [static (Routing Options) 语句 · discard/reject/preference 选项（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/static-edit-routing-options.html)
3. [show route · 优先级数值小者优先（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html)
