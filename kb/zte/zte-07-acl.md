---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: ACL（访问控制列表）
devices: ZXR10 5900/5950/6900/8900 系列交换机、ZXR10 1800/2800 系列路由器等
keywords: [中兴, ZXR10, ACL, acl standard number, acl extended number, rule permit, 反掩码, ip access-group, 标准ACL, 扩展ACL]
sources:
  - https://csl.zte.com.cn/manual_zh/topics/1684391333553.html
  - https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html
  - https://patents.google.com/patent/CN101262503B/en
verified: medium
updated: 2026-10-30
---

# 中兴 · ACL

> ZXR10 的 ACL 分标准（`acl standard number`，按源 IP 过滤）与扩展（`acl extended number`，按协议/源目 IP/端口过滤）两类，思路与 Cisco 一致，但**规则写法有两套**：新平台用 `rule <序号> permit|deny …`，老平台（如 ZXR10 1800）直接在 ACL 视图里写 `permit …`/`deny …`，没有 `rule` 关键字。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建标准 ACL | `acl standard number 10` | 全局配置 `(config)#` | 标准 ACL 编号范围 1–99（部分平台另有扩展段，以官方手册为准） |
| 创建扩展 ACL | `acl extended number 100` | 全局配置 `(config)#` | 扩展 ACL 编号范围 100–199；编号段跨版本可能不同 |
| 加规则（新平台） | `rule 1 permit 10.1.1.0 0.0.0.255` | 标准 ACL 视图 `(config-std-acl)#` | 序号决定匹配顺序，从小到大逐条匹配 |
| 加规则（老平台） | `permit 10.1.1.0 0.0.0.255` | 标准 ACL 视图 | 1800 等平台无 `rule` 关键字，直接写动作（以官方手册为准） |
| 扩展规则（TCP） | `rule 1 permit tcp 10.1.1.0 0.0.0.255 any eq 80` | 扩展 ACL 视图 `(config-ext-acl)#` | 协议 + 源 + 目的 + 端口；`any` 表示任意地址 |
| 扩展规则（ICMP） | `rule 2 permit icmp any any` | 扩展 ACL 视图 | 排障期临时放通 ping，用完及时收敛 |
| 显式拒绝 | `rule 3 deny ip any any` | 扩展 ACL 视图 | 写在末尾即“白名单”效果；是否已有隐式拒绝以官方手册为准 |
| 规则重排 | `move 3 after 1` / `move 3 before 1` | 扩展 ACL 视图 | 插入规则不必删旧建新，避免中间态放通（关键字随版本） |
| 删除单条规则 | `no rule 1`（新）/ `no permit 10.1.1.0 0.0.0.255`（老） | ACL 视图 | 老平台按“动作+地址”整条匹配删除 |
| 应用到接口 | `ip access-group 10 in` | 接口视图 `(config-if)#` | 方向 `in`/`out` 相对设备转发方向，语法以官方手册确认 |
| 查看 ACL 配置 | `show running-config` | 特权视图 `ZXR10#` | 最可靠的核对方式；专门的 `show acl` 命令族随版本差异大 |
| 查看接口应用 | `show running-config interface gei_1/1` | 特权视图 | 确认 `ip access-group` 是否真的挂到了目标接口 |
| 掩码规则 | 反掩码（wildcard） | — | `0.0.0.255` = 匹配一个 /24；与 `ip address` 使用的正掩码相反，最易抄错 |

## 典型示例

```text
ZXR10>enable
ZXR10#configure terminal

// 1) 标准 ACL：只允许管理网段 10.10.10.0/24 访问
ZXR10(config)#acl standard number 10
ZXR10(config-std-acl)#rule 1 permit 10.10.10.0 0.0.0.255
ZXR10(config-std-acl)#rule 2 deny any                          // 末尾显式拒绝，避免依赖隐式行为
ZXR10(config-std-acl)#exit

// 2) 扩展 ACL：放通 Web 与 ICMP，其余拒绝
ZXR10(config)#acl extended number 100
ZXR10(config-ext-acl)#rule 1 permit tcp any 10.20.1.0 0.0.0.255 eq 80
ZXR10(config-ext-acl)#rule 2 permit icmp any any
ZXR10(config-ext-acl)#rule 3 deny ip any any
ZXR10(config-ext-acl)#exit

// 3) 应用到接口（方向与语法以官方手册确认）
ZXR10(config)#interface gei_1/1
ZXR10(config-if)#ip access-group 10 in
ZXR10(config-if)#exit

ZXR10(config)#end
ZXR10#show running-config interface gei_1/1                     // 确认策略真的挂上了
ZXR10#write
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：`rule` 关键字、ACL 视图名（`(config-std-acl)#` / `(config-ext-acl)#`）、`ip access-group` 的可用方向在不同平台/版本上不一致，落配置前先 `show version software` 并在目标机上用 `?` 核对。
- **两套规则语法并存**：老平台（ZXR10 1800 系列文档）写 `permit 10.1.1.0 0.0.0.255`，新平台（5900/8900 命令手册）写 `rule 1 permit 10.1.1.0 0.0.0.255`。抄配置时混用会直接报错，跨平台迁移必须按机型改写。
- 反掩码陷阱：ACL 用反掩码（`0.0.0.255`），而接口 `ip address` 用点分正掩码（`255.255.255.0`），OSPF `network` 又用反掩码——同一份配置里两种掩码并存，写反会把整段网段放通或拒绝。
- 匹配顺序是从小序号到大序号，命中即停；把 `deny any any` 写在前面会让后面的 permit 全部失效。
- 末尾是否有隐式拒绝，官方文档表述不一；生产配置建议显式写拒绝规则，不依赖默认行为。
- ACL 只负责“匹配”，部分平台需要再挂到流策略/策略路由里才真正影响转发（以官方手册为准）；只建 ACL 不应用等于没配。
- 修改在线 ACL 会立即影响转发，部分平台在规则变更瞬间会出现短暂放通/丢包，变更应走维护窗口并先在测试口验证。

## 高危操作 ⚠️

- ⚠️ 在管理接口 `in` 方向误加 `deny any`／错误反掩码会立即自锁失联，操作前必须保留 Console 或另有一条带外管理路径。
- ⚠️ 删除正被接口引用的 ACL（`no acl standard number 10`）会改变该接口的过滤行为，部分版本等效于全部放通，属于隐性安全风险；改前先 `show running-config` 备份。
- ⚠️ 变更前导出 `show running-config`，变更后用 `show running-config interface` 复核策略是否按预期挂载；异常时按备份配置回退，不要在现场试错。

## 参考来源

1. [中兴官方手册站（ZXR10 配置指导，ACL 相关章节，实测可访问）](https://csl.zte.com.cn/manual_zh/topics/1684391333553.html)（官方）
2. [ZXR10 路由交换机基本操作和配置 V2.1 课件（视图层级、`show` 回显与权限说明）](https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html)
3. [中兴专利 CN101262503B（含 VLAN/接口/路由实测配置例，可作为命令形态旁证）](https://patents.google.com/patent/CN101262503B/en)（中兴官方申请文件）
