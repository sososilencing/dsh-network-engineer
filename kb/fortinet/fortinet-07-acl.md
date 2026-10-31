---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: 访问控制（地址对象 + 防火墙策略 + Local-in）
devices: FortiGate 40F/60F/100F/600E/1800F 等
keywords: [ACL, config firewall policy, 隐含拒绝, srcintf dstintf srcaddr dstaddr service, set action accept, set nat enable, config firewall address, service custom, local-in-policy, VIP 端口映射, diagnose firewall iprope]
sources:
  - https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/333889629/config-firewall-policy
  - https://docs.fortinet.com/document/fortigate/6.4.3/cli-reference/248620
  - https://docs.fortinet.com/index.php/document/fortigate/6.4.1/cli-reference/312620/firewall-local-in-policy
  - https://support.fortinet.com.cn/policy_objects/local_traffic_policy/local_in_policy
  - https://support.fortinet.com.cn/policy_objects/single_line_access/static_address_line
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· 访问控制（地址对象 + 策略 + Local-in）

> FortiGate 没有交换机那种"编号 ACL"：**地址对象 + 服务对象 + 防火墙策略**就是 ACL，策略按 ID 从小到大匹配、命中即停，**默认隐含拒绝**（需显式 `set action accept`）。目标是 FortiGate 自身的流量（管理面）走另一套 **Local-in 策略**，而 Local-in 默认隐含**允许**。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 地址对象 | `config firewall address` → `edit "lan"` → `set subnet 192.168.1.0 255.255.255.0` | `(address) #` | 点分掩码；对象名可含 `/`，如 `"117.53.1.0/24"` |
| IP 范围对象 | `set type iprange` + `set start-ip 10.1.1.10` + `set end-ip 10.1.1.20` | `(lan) #` | 默认 `type subnet`，改类型后 `set subnet` 失效 |
| 地域对象 | `set type geography` + `set country "CN"` | `(china) #` | 按国家做源地址控制，常用于限制管理面来源 |
| 地址组 | `config firewall addrgrp` → `edit "grp_lan"` → `set member "lan" "dmz"` | `(addrgrp) #` | 策略里引用组，便于批量维护 |
| 自定义服务 | `config firewall service custom` → `edit "SSL_VPN_10443"` → `set tcp-portrange 10443` | `(custom) #` | 非标准端口必须自建服务对象 |
| 放行策略 | `config firewall policy` → `edit 1` → `set srcintf` / `set dstintf` / `set srcaddr` / `set dstaddr` / `set service` / `set schedule "always"` | `(policy) #` | 五元组必须全部显式配置 |
| 动作与 NAT | `set action accept` + `set nat enable` | `(1) #` | 动作默认 deny；`nat enable` 才把源地址 SNAT 成出接口地址 |
| 显式拒绝 | `set action deny` | `(1) #` | 用于在放行策略之前插一条例外拒绝 |
| 调整顺序 | `move <ID> before <ID>` | `(policy) #` | 策略顺序决定匹配结果（以官方手册为准） |
| 端口映射（DNAT） | `config firewall vip` → `edit "web"` → `set extip` / `set mappedip` / `set portforward enable` / `set extport` / `set mappedport` | `(vip) #` | VIP 做目的地址转换，策略里 `dstaddr` 引用 VIP |
| 管理面 ACL | `config firewall local-in-policy` → `edit 1` → `set intf "port1"` / `set srcaddr` / `set service "SSH"` / `set action accept\|deny` | `(local-in-policy) #` | 只匹配目标是 FortiGate 自身、未穿越设备的流量 |
| 查看内部策略表 | `diagnose firewall iprope list` | 操作模式 | FortiGate 内部实际生效的策略表 |
| 匹配过程可视化 | `diagnose debug flow show iprope enable` + `diagnose debug flow trace start 10` | 操作模式 | 输出里会打印命中的 policy ID 与 drop 原因 |
| 查会话 | `diagnose sys session filter dst 8.8.8.8` / `diagnose sys session list` | 操作模式 | 策略改了不生效时，先确认是不是老会话还在 |

## 典型示例

```text
// 场景：内网 192.168.1.0/24 上网并做 SNAT；同时只允许 10.10.10.0/24 管理 port1 的 SSH
FortiGate # config firewall address
FortiGate (address) # edit "lan"
FortiGate (lan) # set subnet 192.168.1.0 255.255.255.0
FortiGate (lan) # next
FortiGate (address) # end

FortiGate # config firewall policy
FortiGate (policy) # edit 1
FortiGate (1) # set name "to_Internet"
FortiGate (1) # set srcintf "port3"                     // 入接口：内网口
FortiGate (1) # set dstintf "port1"                     // 出接口：WAN
FortiGate (1) # set srcaddr "lan"
FortiGate (1) # set dstaddr "all"
FortiGate (1) # set service "ALL"
FortiGate (1) # set schedule "always"
FortiGate (1) # set action accept                       // 不写 action 就是 deny
FortiGate (1) # set nat enable                          // 源 NAT 成 port1 地址
FortiGate (1) # next
FortiGate (policy) # end

FortiGate # config firewall local-in-policy             // 管理面：只放行管理网段 SSH
FortiGate (local-in-policy) # edit 1
FortiGate (1) # set intf "port1"
FortiGate (1) # set srcaddr "lan"
FortiGate (1) # set dstaddr "all"
FortiGate (1) # set service "SSH"
FortiGate (1) # set action accept
FortiGate (1) # next
FortiGate (local-in-policy) # edit 2
FortiGate (2) # set intf "port1"
FortiGate (2) # set srcaddr "all"
FortiGate (2) # set dstaddr "all"
FortiGate (2) # set service "SSH"
FortiGate (2) # set action deny                         // 兜底拒绝其余来源
FortiGate (2) # next
FortiGate (local-in-policy) # end

FortiGate # diagnose firewall iprope list              // 看真正生效的策略表
FortiGate # diagnose debug flow filter addr 8.8.8.8    // 验证命中哪条策略
FortiGate # diagnose debug flow show iprope enable
FortiGate # diagnose debug flow trace start 10
FortiGate # diagnose debug enable
```

## 陷阱与版本差异

- **默认拒绝方向要分清**：普通防火墙策略隐含 **deny all**（与华为 `traffic-filter` 未匹配默认转发相反）；而 **Local-in 策略隐含 permit all**（官方明确说明），所以管理面做白名单必须自带一条 deny 兜底。
- 五元组缺一不可：`srcintf`/`dstintf`/`srcaddr`/`dstaddr`/`service` 少配一个，策略可能永远匹配不上，表现为"策略明明放行了但流量不通"。
- `set action` 默认 `deny`，必须显式 `accept`；`set nat enable` 才做 SNAT，做 DNAT/端口映射要用 VIP 并把 VIP 放进策略的 `dstaddr`。
- **改策略不刷新已有会话**：老会话仍按旧策略转发，必须 `diagnose sys session filter ...` + `diagnose sys session clear` 或等老化，否则会误判"配置没生效"。
- 策略按 ID 升序匹配、命中即停，插入例外要么用 `move` 调序，要么规划好 ID；`edit 0` 会自动分配一个 ID（通常是当前最大值 +1）。
- Local-in 与 SD-WAN 版本差异：FortiOS 7.4.6 / 7.6.1 起，若接口属于 SD-WAN 区域，Local-in 策略只能引用区域而不能引用成员接口，升级后老策略可能失效需重建。管理面的第一道门其实是接口的 `set allowaccess`，Local-in 是第二道门，只配一个都不完整。

## 高危操作 ⚠️

- ⚠️ 删除或停用承载在线业务的策略会立即断业务，且旧会话可能短时残留，⚠️ 变更前 `execute backup config` 备份，并准备好用 `set status disable` 代替 `delete` 做灰度。
- ⚠️ Local-in 策略误写 `action deny`（尤其针对 `all`/`HTTPS`/`SSH`）会当场锁死管理面，⚠️ 操作前保留 Console 通道，并用最小范围（只针对具体源地址）先验证。
- ⚠️ `set allowaccess` 是覆盖式赋值，漏写 `https`/`ssh` 会立刻失去远程管理；⚠️ 远程修改前先 `append allowaccess ssh` 方式增量调整。
- ⚠️ `diagnose sys session clear` 会中断所有匹配过滤条件的在线会话（业务连接直接断），⚠️ 先在测试条件上 `diagnose sys session list` 确认范围再清。

## 参考来源

1. [config firewall policy（docs.fortinet.com，FortiOS 7.2.14 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/333889629/config-firewall-policy)
2. [firewall address（docs.fortinet.com，FortiOS 6.4.3 CLI Reference）](https://docs.fortinet.com/document/fortigate/6.4.3/cli-reference/248620)
3. [firewall local-in-policy（docs.fortinet.com，FortiOS 6.4.1 CLI Reference）](https://docs.fortinet.com/index.php/document/fortigate/6.4.1/cli-reference/312620/firewall-local-in-policy)
4. [Local-in 策略：隐含 permit all、语法与 SD-WAN 版本差异（FortiGate 中文一本通）](https://support.fortinet.com.cn/policy_objects/local_traffic_policy/local_in_policy)
5. [静态地址线路上网配置：address/policy/SNAT 完整示例（FortiGate 中文一本通）](https://support.fortinet.com.cn/policy_objects/single_line_access/static_address_line)
