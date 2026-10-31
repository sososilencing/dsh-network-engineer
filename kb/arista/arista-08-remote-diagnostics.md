---
vendor: Arista
os: EOS
topic: 远程管理与诊断
devices: vEOS、DCS-7050X/7060X/7280R/7500R、CCS-720XP 等 EOS 设备
keywords: [management ssh, management api http-commands, eAPI, show version, show interfaces status, show ip route, show ip ospf neighbor, show lldp neighbors, show tech-support, bash]
sources:
  - https://www.arista.com/en/um-eos/eos-user-security
  - https://www.arista.com/en/um-eos/eos-acls-and-route-maps
  - https://www.arista.com/en/um-eos/eos-open-shortest-path-first-version-2
verified: high
updated: 2026-10-30
---

# Arista · 远程管理与诊断

> 适用于 EOS 的 SSH/eAPI 管理面配置与常用排障命令。管理命令一应俱全但要记牢：**所有变更最后都要 `copy running-config startup-config`**；`reload now`、`write erase` 属高危。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 启用 SSH 服务 | `management ssh` → `no shutdown` | `(config)#` → `management ssh` 视图 | 远程管理首选 |
| SSH 服务 ACL | `ip access-group ACL in` | `management ssh` 视图 | 只限制 SSH 进程的源地址 |
| 限制 SSH VRF | `vrf MGMT` | `management ssh` 视图 | 把 SSH 绑定到管理 VRF |
| 启用 eAPI | `management api http-commands` → `no shutdown` | `(config)#` → eAPI 视图 | JSON-RPC/HTTP 自动化入口 |
| eAPI 只开 HTTPS | `protocol https` / `no protocol http` | eAPI 视图 | 关闭明文 HTTP；参数以官方手册为准 |
| eAPI 绑定 VRF | `vrf MGMT` | eAPI 视图 | 参数以官方手册为准 |
| 查看版本 | `show version` | `>` 特权 EXEC | EOS 版本、型号、序列号、启动镜像 |
| 接口一览 | `show interfaces status` | `>` 特权 EXEC | 状态/VLAN/速率/双工，排障第一命令 |
| 单口详情 | `show interfaces Ethernet1` | `>` 特权 EXEC | 计数器、错误、MTU、MAC |
| 错误计数 | `show interfaces counters errors` | `>` 特权 EXEC | 定位 CRC/丢包/光模块问题 |
| IP 接口汇总 | `show ip interface brief` | `>` 特权 EXEC | 所有三层口地址与 up/down |
| 路由表 | `show ip route` | `>` 特权 EXEC | `C` 直连、`S` 静态、`O` OSPF |
| ARP 表 | `show ip arp` | `>` 特权 EXEC | IP↔MAC 与出接口对应关系 |
| OSPF 邻居 | `show ip ospf neighbor` | `>` 特权 EXEC | 关注 State=FULL、Dead Time |
| LLDP 邻居 | `show lldp neighbors` | `>` 特权 EXEC | 核对物理连线的对端设备与端口 |
| LLDP 详情 | `show lldp neighbors detail` | `>` 特权 EXEC | 含管理地址、系统描述、能力 |
| MAC 表 | `show mac address-table` | `>` 特权 EXEC | 可加 `interface`/`vlan` 过滤 |
| 日志 | `show logging` / `show logging last 50` | `>` 特权 EXEC | 接口抖动、协议邻居变化都在这里 |
| 在线用户 | `show users` | `>` 特权 EXEC | 变更前确认会话数，避免被人踩 |
| 运行配置 | `show running-config` | `>` 特权 EXEC | 变更前导出留存 |
| 一键收集 | `show tech-support` | `>` 特权 EXEC | 打包给 TAC 的标准输出 |
| Linux Shell | `bash` / `bash sudo` | `>` 特权 EXEC | 进入底层 EOS Linux，慎用 |

## 典型示例

```text
switch# configure terminal
switch(config)# username admin privilege 15 role network-admin secret 0 Arista@2026
switch(config)# management ssh
switch(config-mgmt-ssh)# no shutdown
switch(config-mgmt-ssh)# vrf MGMT
switch(config-mgmt-ssh)# ip access-group SSH-SRC in     // 仅允许运维网段登录
switch(config-mgmt-ssh)# exit
switch(config)# management api http-commands
switch(config-mgmt-api-http-cmds)# no shutdown
switch(config-mgmt-api-http-cmds)# protocol https
switch(config-mgmt-api-http-cmds)# no protocol http
switch(config-mgmt-api-http-cmds)# vrf MGMT
switch(config-mgmt-api-http-cmds)# exit
switch(config)# end
switch# copy running-config startup-config              // ★ 不保存则重启丢失
```

日常巡检/排障：

```text
switch# show version
switch# show interfaces status
switch# show interfaces counters errors
switch# show ip interface brief
switch# show ip route
switch# show ip arp
switch# show ip ospf neighbor
switch# show lldp neighbors
switch# show logging last 50
switch# show tech-support > flash:tech-20261030.txt     // 输出落到 flash 便于导出
```

## 陷阱与版本差异

- 一切命令配完必须 `copy running-config startup-config`；用 `show startup-config` 与 `show running-config` 对拍是确认"已落盘"的最快方法。
- `management ssh`、`management api http-commands` 均为独立子视图，服务启停用其中的 `shutdown`/`no shutdown`；eAPI 的 `protocol`/`vrf` 参数随版本差异较大，落地前 `?` 现场确认（以官方手册为准）。
- 服务 ACL（`management ssh` 下）与 `control-plane` ACL 是两套东西：前者只管 SSH 进程，后者保护整块 CPU 上送流量，别混用。
- `show tech-support` 输出极大，直接刷屏会卡会话；官方建议重定向到文件再导出。
- `bash` 进入的是底层 Linux，命令、路径与 EOS CLI 完全不同；不要在 Shell 里改 EOS 配置文件。
- LLDP 默认开启；若 `show lldp neighbors` 为空，先确认对端口未 `no lldp run` 且对端也启用 LLDP。

## 高危操作 ⚠️

- ⚠️ `reload now`（以及 `reload` 后确认）会**立即重启**，所有未保存配置丢失、业务全断；执行前先 `copy running-config startup-config`，并确认双机/双上行可承载。
- ⚠️ `write erase` / `delete startup-config` 清空启动配置，重启后设备回到出厂状态并**失去全部 IP 与管理入口**；只能在有 Console 与完整备份的前提下操作。
- ⚠️ `no ip routing` 立即中断全部三层转发（SVI、路由口、OSPF 全断），是 EOS 最危险的"一行命令"。
- ⚠️ 在 `management ssh` 下 `shutdown` 或配错服务 ACL，会当场切断远程登录；操作前保留 Console/带外通道，并记住 EOS 默认无 Telnet 兜底。
- 标准变更流程：`show running-config` 导出 → `copy running-config flash:backup-<日期>.cfg` → 变更 → `show ...` 验证 → 异常时 `copy flash:backup-<日期>.cfg running-config` 回退 → 确认后 `copy running-config startup-config`。

## 参考来源

1. [EOS User Manual · User Security（用户名、角色与本地认证）（arista.com）](https://www.arista.com/en/um-eos/eos-user-security)
2. [EOS User Manual · ACLs and Route Maps · Service ACLs / management ssh（arista.com）](https://www.arista.com/en/um-eos/eos-acls-and-route-maps)
3. [EOS User Manual · OSPFv2 · show ip ospf neighbor 输出（arista.com）](https://www.arista.com/en/um-eos/eos-open-shortest-path-first-version-2)
