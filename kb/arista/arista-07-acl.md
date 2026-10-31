---
vendor: Arista
os: EOS
topic: ACL 访问控制
devices: vEOS、DCS-7050X/7060X/7280R/7500R、CCS-720XP 等 EOS 设备
keywords: [ACL, ip access-list, access-group, permit, deny, 序列号, control-plane, 服务 ACL, management ssh, counters per-entry]
sources:
  - https://www.arista.com/en/um-eos/eos-acls-and-route-maps
  - https://www.arista.com/en/um-eos/eos-security
  - https://www.arista.com/en/um-eos/eos-layer-3-configuration
verified: high
updated: 2026-10-30
---

# Arista · ACL 访问控制

> 适用于 EOS 的命名 ACL（IPv4 / IPv6 / MAC）、接口 `ip access-group` 与控制面（control-plane / 服务 ACL）。**EOS 特有坑：ACL 配置视图是"组变更模式"，`exit` 才提交、`abort` 丢弃。**

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 创建 IPv4 ACL | `ip access-list EXT-IN` | `(config)#` | 命名 ACL，进入 `(config-acl-<name>)#` |
| 创建标准 ACL（仅源地址） | `ip access-list standard STD-IN` | `(config)#` | 进入 `(config-std-acl-<name>)#` |
| 创建 IPv6 ACL | `ipv6 access-list V6-IN` | `(config)#` | 关键字为 `ipv6 ... permit ipv6` |
| 创建 MAC ACL | `mac access-list MAC-IN` | `(config)#` | 二层头过滤，需 `mac access-group` 应用 |
| 放行网段 | `permit ip 10.10.10.0/24 any` | ACL 视图 | 掩码用 CIDR 或地址+掩码 |
| 放行主机 | `permit ip any host 10.20.10.1` | ACL 视图 | `any` 表示全部地址 |
| 放行特定应用 | `permit tcp any any eq ssh` | ACL 视图 | 支持 eq/neq/lt/gt/range |
| 显式拒绝 | `deny ip any any log` | ACL 视图 | `log` 命中时打 syslog |
| 结尾放行 | `permit ip any any` | ACL 视图 | **不写则未匹配流量被丢弃** |
| 插入规则（带序号） | `15 permit ip 10.30.10.0/24 host 10.20.10.1` | ACL 视图 | 序号决定顺序，插在两规则之间 |
| 删除某条规则 | `no 20` 或 `no permit ip any host 10.20.10.1` | ACL 视图 | 两种写法等价 |
| 规则重排序 | `resequence 100 20` | ACL 视图 | 首条 100，步长 20 |
| 打开逐条计数 | `counters per-entry` | ACL 视图 | 新 ACL 默认不计数的 |
| 应用到接口 | `ip access-group EXT-IN in` | 接口视图 | 每方向每类型只允许一个 ACL |
| 应用到控制面 | `control-plane` → `ip access-group CP-ACL in` | `(config)#` → `(config-cp)#` | 保护交换机自身 CPU |
| 应用到 SSH 服务 | `management ssh` → `ip access-group ACL in` | `management ssh` 视图 | 服务 ACL，只限制 SSH 进程 |
| 应用到 MAC ACL | `mac access-group MAC-IN in` | 接口视图 | 与 IP ACL 相互独立 |
| 查看全部 ACL | `show ip access-lists` | `>` 特权 EXEC | 含规则、计数器、应用位置 |
| 查看 ACL 清单 | `show ip access-lists summary` | `>` 特权 EXEC | 列出所有 ACL 与规则条数 |
| 清计数器 | `clear ip access-lists counters <name>` | `>` 特权 EXEC | 排障前先清零再看增量 |
| 默认控制面 ACL | `default-control-plane-acl` | — | 只读，删掉自定义 ACL 后自动恢复 |

## 典型示例

```text
switch# configure terminal
switch(config)# ip access-list EXT-IN
switch(config-acl-EXT-IN)# 10 permit tcp 10.10.10.0/24 any eq ssh
switch(config-acl-EXT-IN)# 20 permit ip 10.10.20.0/24 any
switch(config-acl-EXT-IN)# 30 deny ip any any log
switch(config-acl-EXT-IN)# counters per-entry
switch(config-acl-EXT-IN)# show pending          // 预览待提交内容
switch(config-acl-EXT-IN)# exit                  // ★ exit 提交；abort 则丢弃全部修改
switch(config)# interface Ethernet 49
switch(config-if-Et49)# ip access-group EXT-IN in
switch(config-if-Et49)# exit
switch(config)# control-plane
switch(config-cp)# ip access-group CP-ACL in     // 保护管理面
switch(config-cp)# exit
switch(config)# end
switch# copy running-config startup-config       // ★ 不保存则重启丢失
switch# show ip access-lists EXT-IN
switch# show ip access-lists summary
```

## 陷阱与版本差异

- **ACL 视图是组变更模式**（官方原文）：改完必须 `exit` 才写入 running-config；此时用 `show ip access-lists` 看不到新规则是正常现象，用 `show pending` / `show diff` 查看，`abort` 则整批放弃。
- ACL **末尾隐含拒绝**：任何未匹配规则的报文都会被丢弃。写 `deny ip any any` 之后再忘了 `permit ip any any`，会把后续所有流量（含管理流量）一起断掉。
- 接口默认**没有** ACL；一旦 `ip access-group ... in` 生效，方向写错（in/out）等于没生效，EOS 对 out 方向支持还随平台而异，落地前用 `show ip access-lists` 看 `Active on Ingress/Egress`。
- ACL 名字里可以是纯数字（如 `ip access-list standard 10`），但它仍是**命名 ACL**，不要指望 Cisco 那种"数字 ACL 自动归类"的语义。
- 三套 ACL 相互独立：接口 ACL（数据面）、`control-plane` ACL（保护 CPU）、`management ssh` 下的服务 ACL（只限该服务）。改错层等于没做防护。
- `counters per-entry` 是**每个 ACL 级别**的总开关，默认不计；部分平台出方向计数还需 `hardware counter feature acl out ipv4`（以官方手册为准）。

## 高危操作 ⚠️

- ⚠️ 在管理口/上联口 `in` 方向误下 `deny ip any any`，会立即切断 SSH 与全部业务，且无法远程回退；上线前务必保留 Console 会话，并先加 `permit tcp any any eq ssh` 之类保底规则。
- ⚠️ 在 `control-plane` 上误配过严的 ACL 会阻断 OSPF/BGP/SSH/SNMP 等上送 CPU 的协议报文，表现为"接口 up、邻居全 down"，排查成本高。
- ⚠️ `no ip access-group ... in` 与 `default interface ...` 会移除整条 ACL 绑定，等于防火墙失效；变更前 `show running-config interfaces <口>` 留存。
- ⚠️ `no ip access-list <name>` 删除正在被引用的 ACL，绑定点会回落到无 ACL 状态（流量全放行），务必先解绑再删。
- 变更顺序：`show running-config` + `show ip access-lists` 备份 → 变更（用 `show pending` 预演）→ `exit` 提交 → `show ip access-lists` 验证 `Active on` → 异常时 `no ip access-group` 立即解绑 → 确认后 `copy running-config startup-config`。

## 参考来源

1. [EOS User Manual · ACLs and Route Maps（arista.com）](https://www.arista.com/en/um-eos/eos-acls-and-route-maps)
2. [EOS User Manual · Security（arista.com）](https://www.arista.com/en/um-eos/eos-security)
3. [EOS User Manual · Layer 3 Configuration（arista.com）](https://www.arista.com/en/um-eos/eos-layer-3-configuration)
