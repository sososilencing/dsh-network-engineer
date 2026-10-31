---
vendor: 思科（Cisco）
os: NX-OS
topic: 远程管理与诊断
devices: Nexus 3000/5000/7000/9000 系列（mgmt0 带外口 + 带内 inband 管理）
keywords: [SSH, feature ssh, ssh key rsa, telnet server, username role network-admin, logging server, show tech-support, show logging logfile, ethanalyzer, 远程管理]
sources:
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x/m-configuring-ssh-and-telnet.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x.html
  - https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/fundamentals/cisco-nexus-9000-nx-os-fundamentals-configuration-guide-103x.html
verified: high
updated: 2026-10-30
---

# 思科 Cisco NX-OS · 远程管理与诊断

> 适用于 Nexus 的 SSH/Telnet 远程登录、账号与日志服务器配置，以及 `show tech-support` 类排障命令。
> 与 IOS 差异：**SSH 服务默认就是 enabled**（IOS 需 `crypto key generate rsa` 才起），Telnet 服务默认 disabled。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 管理口配 IP | `interface mgmt0` → `ip address 10.99.0.21/24` | `switch(config-if)#` | mgmt0 属 `management` VRF，带外首选 |
| 开/重开 SSH | `feature ssh` | `switch(config)#` | SSH 默认 enabled；被 `no feature ssh` 关掉后才需它重开 |
| 生成主机密钥 | `ssh key rsa 2048` | `switch(config)#` | RSA 位数范围 768–2048，缺省 1024；换钥加 `force` |
| 其他密钥类型 | `ssh key ecdsa 256` / `ssh key dsa` | `switch(config)#` | 用 DSA 时官方要求另配 `ssh keytypes all` 与 `ssh kexalgos all` |
| 建本地账号 | `username netops password 0 Cisco@123 role network-admin` | `switch(config)#` | 角色 `network-admin`（管理）/`network-operator`（只读） |
| 开 Telnet 服务 | `feature telnet` | `switch(config)#` | Telnet 服务**默认关闭**，端口 23；生产环境不建议开 |
| 限制并发会话 | `line vty` → `session-limit 4` | `switch(config-line)#` | 缺省会话数随平台，按官方手册核对 |
| 登录超时 | `line vty` → `exec-timeout 10` | `switch(config-line)#` | 分钟为单位，0 表示永不超时（不建议） |
| 日志服务器 | `logging server 10.99.0.50 6 use-vrf management` | `switch(config)#` | 带外网段必须带 `use-vrf management`，否则发不出去 |
| 实时看日志 | `terminal monitor` | `switch#` | 把日志输出到当前会话，退出即失效 |
| 看日志文件 | `show logging logfile` / `show logging nvram` | `switch#` | 排障必看的两个历史日志来源 |
| 备份配置 | `copy running-config bootflash:backup-20261030.cfg` | `switch#` | 变更前留档，可 `copy bootflash:... running-config` 回灌 |
| 看在线用户 | `show users` | `switch#` | 谁在登、从哪个 IP、空闲多久 |
| 看硬件与版本 | `show version` / `show module` / `show inventory` | `switch#` | 基线采集三件套 |
| 看环境 | `show environment` / `show system resources` | `switch#` | 温度、电源、风扇、CPU/内存 |
| 抓包 | `ethanalyzer local interface inband ...` | `switch#` | CPU 侧抓包；精确语法随版本，以官方手册为准 |
| 一键体检 | `show tech-support` / `show tech-support details` | `switch#` | 开 TAC 前必收；`details` 体量大，注意 SSH 超时 |
| 看诊断结果 | `show diagnostic result module all detail` | `switch#` | 硬件自检，配合 `show license usage` 一起交 |

## 典型示例

```text
switch# configure terminal
switch(config)# interface mgmt0
switch(config-if)# ip address 10.99.0.21/24            // 带外管理地址（management VRF）
switch(config-if)# no shutdown
switch(config-if)# exit
switch(config)# feature ssh                             // 默认已开，此处确保开启
switch(config)# ssh key rsa 2048                        // 缺省 1024 位，安全基线建议 2048
switch(config)# username netops password 0 Cisco@123 role network-admin
switch(config)# line vty
switch(config-line)# session-limit 4
switch(config-line)# exec-timeout 10
switch(config-line)# exit
switch(config)# logging server 10.99.0.50 6 use-vrf management
switch(config)# end
switch# show ssh server                                 // 确认 SSH 端口/密钥状态（命令随版本，以官方手册为准）
switch# show logging logfile
switch# copy running-config bootflash:backup-20261030.cfg
switch# copy running-config startup-config              // 落盘，否则重启即丢
switch# ping 10.99.0.50 vrf management                  // 跨 VRF 测试要带 vrf 关键字
```

## 陷阱与版本差异

- **SSH 默认 enabled**（官方默认值表：SSH server Enabled / Telnet server Disabled / Max SSH login attempts 3 / 默认密钥 RSA 1024 位）。所以"新设备起不来 SSH"通常不是没开特性，而是管理网段不通或 `mgmt0` 未配 IP。
- `no feature ssh` **不关闭 22 端口**：官方说明端口始终开放，系统改为下发 deny 规则拒绝所有外部连接——看起来像"端口通但连不上"，实际是被策略拦掉。
- 删掉全部 SSH 密钥后**无法再启动 SSH 服务**（官方 Caution），换钥请用 `ssh key ... force` 覆盖而不是先删。
- NX-OS **只支持 SSHv2**，SSLv3 因 POODLE 漏洞不再支持；老客户端强制 SSHv1 时连接直接失败。
- `show tech-support` 耗时长，官方建议把 SSH 超时设为 0（无限）再采集，否则会看到 `%VSHD-2-VSHD_SYSLOG_EOL_ERR`。
- mgmt0 在 `management` VRF：`ping`、`traceroute`、`logging server`、`ntp server` 都需带 `use-vrf management` 或 `vrf management`，否则从 default VRF 发出而不可达。
- 本地账号密码用 `password 0` 表示明文输入（系统会自动转存为加密形式）；`role` 关键字随版本支持情况不同，早期版本用 `username ... password ...` 后在 AAA 侧授权，以官方手册为准。

## 高危操作 ⚠️

- ⚠️ `no feature ssh` 或清空 SSH 密钥会让远程管理立即中断（前者下发 deny 规则、后者起不来服务），必须在 Console 或带外通道可用时操作。
- ⚠️ 改 `mgmt0` 的 IP/掩码、或在 `line vty` 下发 `exec-timeout 0`/`session-limit 1` 组合，都可能把自己锁在门外；变更前先 `show users` 确认已有带外会话。
- ⚠️ 关闭/删除本地 `network-admin` 账号且 AAA 服务器不可达时，等于永久失去管理入口；务必保留一个应急本地账号并定期验证。
- ⚠️ `reload`/`write erase` 前先 `copy running-config bootflash:<file>` 备份：`write erase` 会清掉管理 IP 与主机密钥，重启后只能 Console 重配。

## 参考来源

1. [Cisco Nexus 9000 Series NX-OS Security Configuration Guide, Release 10.3(x) · Configuring SSH and Telnet（cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x/m-configuring-ssh-and-telnet.html)
2. [Cisco Nexus 9000 Series NX-OS Security Configuration Guide, Release 10.3(x)（含 User Accounts and RBAC 章节，cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/security/cisco-nexus-9000-nx-os-security-configuration-guide-103x.html)
3. [Cisco Nexus 9000 Series NX-OS Fundamentals Configuration Guide, Release 10.3(x)（含配置文件管理与 `show tech` 采集，cisco.com）](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/fundamentals/cisco-nexus-9000-nx-os-fundamentals-configuration-guide-103x.html)
