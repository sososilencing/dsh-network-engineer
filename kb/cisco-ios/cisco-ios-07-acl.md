---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: ACL 与 NAT
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [ACL, ip access-list standard, ip access-list extended, ip access-group, 隐式 deny, 通配符掩码, NAT, ip nat inside source list, overload, show ip nat translations]
sources:
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/15-mt/sec-data-acl-15-mt-book/sec-create-ip-apply.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/ip/b_176_ip_9300_cg/configuring_network_address_translation.html
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/15-mt/sec-data-acl-15-mt-book.html
verified: high
updated: 2026-10-30
---

# 思科 · ACL 与 NAT

> 适用于 IOS/IOS-XE 的流量过滤与地址转换：ACL 用**通配符掩码（反掩码）**、末尾有**隐式 deny any**，创建后必须应用（`ip access-group` / `access-class` / NAT 引用）才生效；出口做 PAT 常用 `ip nat inside source list ... overload`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 命名标准 ACL | `ip access-list standard MGMT` | `(config)#` | 进入 `(config-std-nacl)#`，只匹配源地址 |
| 命名扩展 ACL | `ip access-list extended WEB-FILTER` | `(config)#` | 进入 `(config-ext-nacl)#`，可匹配源/目的/协议/端口 |
| 编号标准 ACL | `access-list 1 permit 172.16.5.22 0.0.0.0` | `(config)#` | 标准编号 1–99、1300–1999 |
| 编号扩展 ACL | `access-list 100 permit ip 10.1.20.0 0.0.0.255 any` | `(config)#` | 扩展编号 100–199、2000–2699 |
| 添加备注 | `remark Only-Office-Subnet` | `(config-std-nacl)#` | remark 可前可后，最长 100 字符 |
| 拒绝并记录 | `deny ip any any log` | `(config-ext-nacl)#` | 显式 deny 才有命中计数与日志 |
| 应用到接口 | `ip access-group WEB-FILTER in` | `(config-if)#` | `in` 在路由查找前过滤，`out` 在其后 |
| 应用到 vty | `access-class MGMT in` | `(config-line)#` | 限制可登录管理的主机 |
| 查看 ACL 与命中 | `show ip access-list` | 特权 EXEC `#` | 每行前括号内为匹配报文数 |
| 标记 NAT 内网侧 | `ip nat inside` | `(config-if)#` | 内网接口必须标记 |
| 标记 NAT 外网侧 | `ip nat outside` | `(config-if)#` | 公网接口必须标记，否则不做转换 |
| PAT 复用接口地址 | `ip nat inside source list 1 interface gigabitethernet0/0/1 overload` | `(config)#` | 最常见出口写法：ACL 1 指定内网源 |
| 静态一对一 NAT | `ip nat inside source static 10.1.20.10 203.0.113.10` | `(config)#` | 服务器对外发布用 |
| 地址池复用 | `ip nat pool PUBLIC 203.0.113.10 203.0.113.20 netmask 255.255.255.0` | `(config)#` | 配合 `ip nat inside source list 1 pool PUBLIC overload` |
| 查看转换表 | `show ip nat translations` | 特权 EXEC `#` | 核对 inside local/global 映射是否建立 |
| 查看 NAT 统计 | `show ip nat statistics` | 特权 EXEC `#` | 命中/未命中次数，判断规则是否生效 |
| 清空转换表 | `clear ip nat translation *` | 特权 EXEC `#` | ⚠️ 断已有会话，见高危节 |

## 典型示例

```text
R1# configure terminal
R1(config)# ip access-list extended WEB-FILTER
R1(config-ext-nacl)# remark Allow HTTP/HTTPS to server
R1(config-ext-nacl)# permit tcp 10.1.20.0 0.0.0.255 host 203.0.113.10 eq 443
R1(config-ext-nacl)# deny ip any any log                  // 显式 deny，便于看命中计数
R1(config-ext-nacl)# exit
R1(config)# ip access-list standard NAT-INSIDE
R1(config-std-nacl)# permit 10.1.20.0 0.0.0.255           // 反掩码：/24 对应 0.0.0.255
R1(config-std-nacl)# exit
R1(config)# interface gigabitethernet 0/0/1
R1(config-if)# ip address 203.0.113.2 255.255.255.0
R1(config-if)# ip nat outside
R1(config-if)# interface gigabitethernet 0/0/2
R1(config-if)# ip address 10.1.20.254 255.255.255.0
R1(config-if)# ip nat inside
R1(config-if)# ip access-group WEB-FILTER in              // 先建 ACL 再挂接口
R1(config-if)# exit
R1(config)# ip nat inside source list NAT-INSIDE interface gigabitethernet 0/0/1 overload
R1(config)# end
R1# show ip access-list
R1# show ip nat translations
R1# show ip nat statistics
```

## 陷阱与版本差异

- 所有 ACL 末尾都有**隐式 deny any**，只写 permit 就会把其余流量全挡；官方建议显式写 `deny ip any any`，且只有显式 deny 才有计数可查。
- **先建 ACL 再应用到接口**：官方提示，若先挂一个不存在的 ACL 再逐条写，第一条语句立刻生效，紧随其后的隐式 deny 会当场切断业务。
- 编号 ACL（`access-list 1 ...`）**不能删除单条**，`no` 一条等于清掉整个列表；需要可维护性请用命名 ACL（`ip access-list standard/extended`）。
- ACL 掩码是**反掩码/wildcard**：`/24` 写 `0.0.0.255`；写正掩码会匹配到错误范围，与 OSPF `network` 的反掩码习惯一致但与华为 ACL 的正掩码相反。
- Catalyst 9000 NAT 有限制（官方 Limitations）：NAT ACL 不支持显式 deny 条目、NAT 与 PBR 不能配在同一接口、port-channel 不支持 NAT、不支持分片报文转换；换平台前先核对手册。

## 高危操作 ⚠️

- ⚠️ 在 vty 上配错 `access-class`：远程登录被立即拒绝，只能靠 Console 救回；变更前先 `show run | include access-class` 确认 ACL 内容，并保留 Console 通道。
- ⚠️ 对生产接口挂 `ip access-group ... in` 且规则写错（漏 permit 管理网段）：业务与远程管理同时中断；先在测试口验证命中计数，再上生产。
- ⚠️ `clear ip nat translation *`：清空全部 NAT 会话，正在通信的连接全部断开（业务需重连）；只在必要时点对点清理单条转换。
- ⚠️ 删除或改写正在使用的 NAT 规则（`no ip nat inside source ...`）：新连接无法建立转换，表现为"路由通但公网不通"；变更后必须 `show ip nat translations` 验证。

## 参考来源

1. [Creating an IP Access List and Applying It to an Interface（ACL 创建/应用/最佳实践）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/15-mt/sec-data-acl-15-mt-book/sec-create-ip-apply.html)
2. [Configuring Network Address Translation（PAT/overload、NAT 限制）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/ip/b_176_ip_9300_cg/configuring_network_address_translation.html)
3. [Security Configuration Guide: Access Control Lists, Cisco IOS 15M&T（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_data_acl/configuration/15-mt/sec-data-acl-15-mt-book.html)
