---
vendor: 瞻博（Juniper）
os: Junos
topic: 远程管理与诊断
devices: EX2300/EX4300 交换机、MX 系列路由器、SRX 系列防火墙
keywords: [SSH, system services ssh, root-login, system login user, 排障, monitor traffic, show log messages, request support information, 重启, zeroize]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/getting-started.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-zeroize.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· 远程管理与诊断

> Junos 的 SSH **不需要生成密钥**：`set system services ssh` 一提交，设备即用内置主机密钥提供 SSH 服务（没有 Cisco 的 `crypto key generate rsa` 这一步）。账号体系在 `[edit system login]`，排障命令以 `show` 为主，抓包用 `monitor traffic`，一键收集用 `request support information`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 启用 SSH | `set system services ssh` | `[edit]` | 提交后立即可用，无需生成密钥 |
| 禁止 root 走 SSH | `set system services ssh root-login deny` | `[edit]` | 缺省即 deny；`allow` 才放开（风险高） |
| 仅允许 SSHv2 | `set system services ssh protocol-version v2` | `[edit]` | 默认已禁用 v1 的新版本可省略 |
| 关闭 Telnet | `delete system services telnet` | `[edit]` | 缺省未开启，确认没有残留配置 |
| 新建管理账号 | `set system login user ops class super-user authentication plain-text-password` | `[edit]` | 提交时交互输入密码 |
| 配置 SSH 公钥登录 | `set system login user ops authentication ssh-rsa "ssh-rsa AAAA…"` | `[edit]` | 密钥在本地 `ssh-keygen` 生成后粘贴 |
| root 密码/公钥 | `set system root-authentication plain-text-password` / `... ssh-rsa "…"` | `[edit]` | 首次 commit 强制要求设置 root 密码 |
| 自定义权限类 | `set system login class helpdesk permissions view` | `[edit]` | 权限粒度比 Cisco privilege 更细 |
| 登录提示语 | `set system login message "Authorized access only"` | `[edit]` | 合规要求常用 |
| 查看在线用户 | `show system users` | 操作模式 | 谁在线、从哪个源 IP 登录 |
| 查看版本/运行时长 | `show version` / `show system uptime` | 操作模式 | 报障第一手信息 |
| 查看告警 | `show chassis alarms` | 操作模式 | 硬件/风扇/电源告警 |
| 看日志 | `show log messages \| last 50` | 操作模式 | 系统主日志 |
| 搜日志关键字 | `show log messages \| match "error\|down"` | 操作模式 | 排障最常用组合 |
| 抓包 | `monitor traffic interface ge-0/0/1 no-resolve` | 操作模式 | 类似 tcpdump，Ctrl+C 停止 |
| 连通性测试 | `ping 10.0.0.1 count 5 source 1.1.1.1` / `traceroute 10.0.0.1` | 操作模式 | 指定源地址验证回程路径 |
| 一键收集 | `request support information` | 操作模式 | 汇总配置/日志/状态，提工单用 |
| 备份到备用分区 | `request system snapshot` | 操作模式 | 同步 /config 到 /altconfig |

## 典型示例

```text
user@EX-01> configure
[edit]
user@EX-01# set system services ssh                                  // 启用 SSH，无需 crypto key
user@EX-01# set system services ssh root-login deny                  // 禁止 root 直连
user@EX-01# set system login user ops class super-user authentication plain-text-password
New password:
Retype new password:
user@EX-01# set system login user ops authentication ssh-rsa "ssh-rsa AAAAB3Nza… ops@bastion"   // 公钥登录
user@EX-01# set system login message "Authorized access only"
user@EX-01# delete system services telnet                            // 确认未开明文管理
user@EX-01# commit confirmed 5                                       // 改管理面必须带确认
commit complete
user@EX-01# run show system users                                    // 另开一个会话验证能登录再确认
user@EX-01> show log messages | last 30
user@EX-01> monitor traffic interface ge-0/0/1 no-resolve size 1500   // 抓包定位，Ctrl+C 结束
user@EX-01> ping 10.1.12.2 count 5 source 10.1.12.1
user@EX-01> request support information | save /var/tmp/R1-support.txt
```

## 陷阱与版本差异

- **没有 `crypto key generate`**：Junos 的 SSH 服务提交即用，主机密钥由系统自维护；要从 Cisco 迁移过来时不要苦找生成命令。
- root 账号默认无密码且**不能 Telnet 登录**；只有 `set system services ssh root-login allow` 才允许 root 走 SSH，生产网建议保持 deny 并用普通账号 + `class super-user`。
- 改 SSH/账号/管理接口相关的配置后，务必**另开一个会话验证登录**再执行最终 `commit`；只靠当前会话确认等于没确认。
- `plain-text-password` 在 commit 时交互录入并立即加密存储；脚本化场景应改用 `encrypted-password` 或公钥认证，避免口令落到脚本里。
- `monitor traffic` 会占用终端且在高流量口产生大量输出，务必带 `no-resolve`、`size` 等过滤条件并限时终止；不要在业务高峰期对万兆口裸跑。
- `request support information` 输出很大，务必 `| save` 到 `/var/tmp` 再取回，直接刷屏会把当前会话卡住。
- EX 与 MX/SRX 的差异：交换机的 `show chassis alarms`/环境类命令字段不同；SRX 还需看 `show security flow session` 之类安全专有命令（超出本文范围，以官方手册为准）。
- Telnet 若被显式开启（`set system services telnet`），口令明文过网；检查时用 `show configuration system services | display set` 一次看全。

## 高危操作 ⚠️

- ⚠️ `request system reboot`：中断业务，重启前必须 `commit`、`save`，必要时 `request system snapshot`；重启后配置回滚到上次 commit 的状态。
- ⚠️ `request system zeroize`（可选 `media`）：清空配置、密钥、日志并重启，重启后管理口不可达，只能 Console 以 root 登录；SRX 带 VM Host 的机型改用 `request vmhost zeroize`。
- ⚠️ `delete system services ssh` 或把 `root-login`/账号权限改错：会立即失去远程管理，必须 `commit confirmed N` 且保留一个已登录会话。
- ⚠️ `request system power-off` / 断电源：非计划下电可能损坏文件系统，配置与日志有丢失风险，只能在维护窗口执行。

## 参考来源

1. [Getting Started: A Quick Tour of the CLI · 账号、root-login、rollback（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/getting-started.html)
2. [request system zeroize（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/request-system-zeroize.html)
3. [show route · 排障时的路由核对（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli-reference/topics/ref/command/show-route.html)
