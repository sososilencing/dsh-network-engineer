---
vendor: 思科（Cisco）
os: NX-OS
topic: ACL
devices: Nexus 3000/5000/7000/9000 系列（TCAM 容量与 region 划分随平台不同）
keywords: [ACL, ip access-list 命名式, statistics per-entry, ip access-group, ip port access-group, VACL, vlan access-map, 隐式 deny ip any any, TCAM carving]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x/m-configuring-ip-acls.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · ACL

> 适用于 Nexus 的 IPv4/IPv6 ACL：命名式 ACL、Router ACL、Port ACL、VLAN ACL(VACL) 与逐条统计。
> 与 IOS 差异：**不做 `standard/extended` 区分**，命名 ACL 直接支持五元组；末尾隐含 `deny ip any any`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 建命名 ACL | `ip access-list INGRESS-FILTER` | `switch(config)#` | NX-OS 只有命名式（无编号 ACL）；进入 `switch(config-ip-acl)#` |
| 加规则（带序号） | `10 permit tcp 10.1.1.0/24 any eq 443` | `switch(config-ip-acl)#` | 源/目的用 `前缀/长度` + `any`/`host`；序号决定匹配顺序 |
| 插入规则 | `15 deny ip 10.1.1.0/24 10.2.2.0/24` | `switch(config-ip-acl)#` | 在两序号之间插入，避免重排整张表 |
| 删除单条 | `no 15` | `switch(config-ip-acl)#` | 用序号删，不用重写整条规则 |
| 逐条统计 | `statistics per-entry` | `switch(config-ip-acl)#` | 开启后 `show ip access-lists` 每行带命中计数（命令形态随版本，以官方手册为准） |
| 应用到三层口 | `ip access-group INGRESS-FILTER in` | `switch(config-if)#` | Router ACL；支持 `in`/`out`，作用于 SVI、三层物理口、三层 port-channel |
| 应用到二层口 | `ip port access-group PORT-FILTER in` | `switch(config-if)#` | Port ACL；Trunk 口上会对该口所有 VLAN 生效，仅支持 `in` |
| 定义 VACL | `vlan access-map VACL-WEB 10` → `match ip address ACL-WEB` → `action drop` | `switch(config)#` | VLAN 内/跨 VLAN 过滤；`action` 为 `drop`/`forward` |
| 挂载 VACL | `vlan filter VACL-WEB vlan 10` | `switch(config)#` | 一个 VACL 可挂多个 VLAN |
| 查看 ACL | `show ip access-lists` / `show ip access-lists INGRESS-FILTER` | `switch#` | 带统计时每行显示命中次数 |
| 清统计计数 | `clear ip access-list counters INGRESS-FILTER` | `switch#` | 排障时先清零再观察增量 |
| 调整 TCAM 分区 ⚠️ | `hardware access-list tcam region <region> <size>` | `switch(config)#` | 改后**必须 reload** 才生效，超范围会导致 ACL 下发失败 |
| 看 TCAM 使用 | `show system internal access-list resource utilization` | `switch#` | 判断 ACL 条目是否已耗尽 TCAM（命令随平台，以官方手册为准） |

## 典型示例

```text
switch# configure terminal
switch(config)# ip access-list INGRESS-FILTER
switch(config-ip-acl)# statistics per-entry                      // 先开逐条统计，便于验证
switch(config-ip-acl)# 10 permit tcp 10.1.1.0/24 any eq 443       // 放通业务网段到任意目的的 HTTPS
switch(config-ip-acl)# 20 permit tcp 10.1.1.0/24 any eq 22        // 放通 SSH
switch(config-ip-acl)# 30 permit icmp 10.1.1.0/24 any             // 放通 ping，便于排障
switch(config-ip-acl)# 40 deny ip any any log                     // 显式拒绝并记日志（末尾本就有隐式 deny）
switch(config-ip-acl)# exit
switch(config)# interface vlan 10
switch(config-if)# ip access-group INGRESS-FILTER in              // Router ACL 挂到 SVI 入方向
switch(config-if)# exit
switch(config)# interface Ethernet1/10
switch(config-if)# ip port access-group PORT-FILTER in            // Port ACL 挂到二层接入口
switch(config-if)# exit
switch# show ip access-lists INGRESS-FILTER
switch# copy running-config startup-config
```

## 陷阱与版本差异

- **隐式拒绝**：所有 IPv4 ACL 末尾都有不出现在配置里的 `deny ip any any`（IPv6 为 `deny ipv6 any any`），官方明确该隐式规则**不计入逐条统计**。放通型 ACL 必须显式写 permit，否则"配了 ACL 就全断"。
- IPv6 ACL 的隐式拒绝会连 **ND（RS/RA）** 一起挡掉，官方要求在部分平台显式加 `permit icmp any any router-advertisement` 与 `permit icmp any any router-solicitation`。
- 应用位置决定行为：同一份 ACL 命中顺序为 Port ACL → 入向 VACL → 入向 Router ACL → 入向 VTY ACL → 出向 VTY ACL → 出向 Router ACL → 出向 VACL；报文在 VLAN 内二层桥接时**不经过 Router ACL**。
- Port ACL 只支持 `in`，且 Trunk 口上对该口承载的所有 VLAN 生效；要按 VLAN 精细过滤得用 VACL。
- TCAM 是硬约束：ACL 条目数超 region 容量时下发会失败（表现为 `show ip access-lists` 有配置但 `show ip access-group` 不生效），需 `hardware access-list tcam region` 重新划分并 **reload**（部分平台还要求先 `hardware access-list tcam region ...` 顺序配置，参考官方 TCAM carving 说明）。
- 未定义的 ACL 名被应用到接口时，系统按"空 ACL"处理（等同全放通），写错名字不会报错——上线后必须用 `show ip access-lists` 确认名字与计数。
- NAT：NX-OS 的 NAT 章节在 Interfaces 配置指南内（Configuring Static and Dynamic NAT Translation），但支持平台、使能方式与命令集差异很大，务必查对应平台手册，不要照抄 IOS 的 `ip nat inside source list ... overload`。

## 高危操作 ⚠️

- ⚠️ 在管理 SVI 或上联口入方向误挂 `deny ip any any` 类 ACL 会立刻切断远程管理，需保留带外/Console 通道，并准备好 `no ip access-group ... in` 的回退命令。
- ⚠️ 修改已在接口生效的 ACL 是"热生效"的：加一条 deny 立即影响在线流量，务必先 `show ip access-lists` 备份原规则，并按序号插入而非重写。
- ⚠️ `hardware access-list tcam region` 改完必须 `reload` 才生效，且 reload 期间业务全断；region 值设置不当可能导致其他特性（QoS/SPAN/PBR）无法下发。
- ⚠️ 先 `no ip access-group` 还是先删 ACL 有讲究：直接 `no ip access-list <name>` 在部分版本会连带清理接口引用，变更顺序建议"先解绑 → 改 ACL → 再绑定"。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Security Configuration Guide, Release 10.3(x) · Configure IP ACLs（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x/m-configuring-ip-acls.html)
2. [Cisco Nexus 9000 Series NX-OS Security Configuration Guide, Release 10.3(x)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x.html)
3. [Cisco Nexus 9000 Series NX-OS Interfaces Configuration Guide, Release 10.3(x)（含 NAT 章节，cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x.html)
