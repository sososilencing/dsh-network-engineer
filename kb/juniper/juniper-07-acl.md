---
vendor: 瞻博（Juniper）
os: Junos
topic: ACL 与防火墙过滤器
devices: EX2300/EX4300 交换机、MX 系列路由器、SRX 系列防火墙
keywords: [ACL, firewall filter, family inet filter, term, from, then, discard, accept, count, policer, 无状态, 隐式拒绝]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/filter-edit-firewall.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/family-ethernet-switching-edit-interfaces-qfx-series-els.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· ACL 与防火墙过滤器

> Junos 没有 Cisco 那种 `access-list 101 permit tcp ...` 的「ACL」，对应物是**无状态防火墙过滤器**：`firewall family inet filter <名> term <名> from … then …`，定义完再挂到接口方向 (`family inet filter input F`)。因此它可以同时做匹配+计数+限速+改优先级，代价是不具备状态跟踪——真正要「放行回程流量」，SRX 上应使用 security policy 而非过滤器。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 建过滤器与 term | `set firewall family inet filter F-IN term 10` | `[edit]` | term 名是字符串，按书写顺序匹配 |
| 匹配源地址 | `... term 10 from source-address 192.168.10.0/24` | `[edit]` | 必须带前缀长度，无「反掩码」写法 |
| 匹配目的地址 | `... term 10 from destination-address 10.1.1.0/24` | `[edit]` | 同理 |
| 匹配协议 | `... term 10 from protocol tcp` | `[edit]` | `tcp`/`udp`/`icmp`/`ospf` 等，或协议号 |
| 匹配端口 | `... term 10 from destination-port 443` | `[edit]` | 需先限定 `protocol tcp/udp` |
| 匹配 ICMP 类型 | `... term 20 from icmp-type echo-request` | `[edit]` | — |
| 放行 | `... term 10 then accept` | `[edit]` | 终止动作 |
| 丢弃 | `... term 30 then discard` | `[edit]` | 静默丢弃，不回 ICMP |
| 拒绝 | `... term 30 then reject` | `[edit]` | 回 ICMP 不可达，便于定位 |
| 计数 | `... term 10 then count HIT-10` | `[edit]` | 用 `show firewall` 看命中数 |
| 限速 | `... term 40 then policer POLICE-10M` | `[edit]` | 需先定义 `firewall policer` |
| 定义 policer | `set firewall policer POLICE-10M if-exceeding bandwidth-limit 10m burst-size-limit 100k then discard` | `[edit]` | 令牌桶限速（以官方手册为准） |
| 挂到接口入向 | `set interfaces ge-0/0/1 unit 0 family inet filter input F-IN` | `[edit]` | 方向为 `input`/`output` |
| 保护管理面 | `set interfaces lo0 unit 0 family inet filter input PROTECT` | `[edit]` | 限制 SSH/SNMP 来源的标准做法 |
| 查看命中计数 | `show firewall` | 操作模式 | 按过滤器/term 显示字节与包数 |
| 查看过滤器配置 | `show configuration firewall \| display set` | 操作模式 | 输出 set 形式便于评审 |
| 自测匹配 | `show firewall filter F-IN` | 操作模式 | 只看某个过滤器的计数 |

## 典型示例

```text
user@R1> configure
[edit]
user@R1# set firewall family inet filter F-IN term 10 from source-address 192.168.10.0/24   // 办公网
user@R1# set firewall family inet filter F-IN term 10 from protocol tcp
user@R1# set firewall family inet filter F-IN term 10 from destination-port 443
user@R1# set firewall family inet filter F-IN term 10 then count WEB-HIT
user@R1# set firewall family inet filter F-IN term 10 then accept
user@R1# set firewall family inet filter F-IN term 20 from protocol tcp
user@R1# set firewall family inet filter F-IN term 20 from destination-port 22
user@R1# set firewall family inet filter F-IN term 20 then discard                  // 其余来源禁 SSH
user@R1# set firewall family inet filter F-IN term 30 from protocol icmp
user@R1# set firewall family inet filter F-IN term 30 then accept                   // 允许 ping
user@R1# set firewall family inet filter F-IN term 90 then accept                    // 末尾显式放行（默认拒绝）
user@R1# set interfaces ge-0/0/1 unit 0 family inet filter input F-IN                // 挂到入向
user@R1# commit confirmed 5
commit complete
user@R1# run show firewall filter F-IN      // 看各 term 命中，确认规则顺序符合预期
user@R1# commit                                 // 确认
```

## 陷阱与版本差异

- **与「ACL」概念的最大差别**：过滤器是**无状态**的，只匹配单向报文。放行 TCP 443 入向不会自动放行回程；有状态需求（session、ALG、NAT）在 SRX 上要用 `security policies`，过滤器只做补充。
- 过滤器**末尾没有隐式 accept**：未匹配任何 term 的报文被丢弃（部分平台会提示 implicit discard 计数），所以最后务必写一条明确的 `then accept` 或确认就是要全丢。
- term 名（如 `10`）只是标识，**匹配顺序按配置中书写/插入顺序**，不是按名字大小；插入位置要用 `insert` 控制，乱序会导致策略不生效。
- `discard` 与 `reject` 的差异在排障时很关键：`discard` 让源端只看到超时，容易被误判为路由问题；`reject` 会回 ICMP 不可达。
- 平台能力差异明显：EX 系列多为纯 ingress 过滤（output 方向支持受限），MX 支持 inet/inet6 双向并有更多匹配字段；跨平台复用同一份过滤器前需确认该机型支持哪些 match 条件（以官方手册为准）。
- EX 上还可按 VLAN/端口挂过滤器（`family ethernet-switching filter`），与 `family inet filter` 是三层的不同挂载点，二层过滤不要挂错层。
- `policer` 的 `bandwidth-limit` 单位与生效粒度随平台/版本而异，配置后必须用 `show firewall` 与实测打流验证（以官方手册为准）。

## 高危操作 ⚠️

- ⚠️ 在管理接口/`lo0` 上挂入向过滤器：一旦误拦 SSH/SNMP 源地址，立刻失去远程管理能力，必须 `commit confirmed N` 兜底。
- ⚠️ 漏写末尾 `then accept`：整段流量被隐式丢弃，表现为「一挂过滤器全网不通」。
- ⚠️ term 顺序写反（先 `discard` 后 `accept`）：后写的 accept 永远不被命中，策略形同虚设且难查。
- ⚠️ 大量 `policer` 限速值调低：会直接压垮业务吞吐且只在打流时暴露，变更前先评估峰值。

## 参考来源

1. [filter (Configuring) 语句 · [edit firewall] 层级与权限（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/filter-edit-firewall.html)
2. [CLI Configuration Mode Overview · 层级路径与 set 语法（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html)
3. [ethernet-switching（family）语句 · 二层与三层挂载点区分（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/statement/family-ethernet-switching-edit-interfaces-qfx-series-els.html)
