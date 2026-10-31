---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: 远程管理与诊断
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [SSH, crypto key generate rsa, ip domain-name, line vty, transport input ssh, username privilege 15, access-class, show ip interface brief, show logging, debug]
sources:
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_ssh/configuration/15-mt/sec-usr-ssh-15-mt-book/sec-secure-shell-v2.html
  - https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_ssh/configuration/15-mt/sec-usr-ssh-15-mt-book/sec-ssh-term-line.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg/configuring_system_message_logs.html
verified: high
updated: 2026-10-30
---

# 思科 · 远程管理与诊断

> 适用于 IOS/IOS-XE 的 SSH 登录加固与日常排障。**关键顺序**：`hostname` + `ip domain-name` → `crypto key generate rsa modulus 2048` → `ip ssh version 2` → `line vty` + `transport input ssh` + `login local`；顺序颠倒会生成不了密钥或直接把自己关在门外。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 设置域名 | `ip domain-name example.com` | `(config)#` | 与 hostname 一起决定 RSA 密钥标签，必须先配 |
| 生成 RSA 密钥 | `crypto key generate rsa modulus 2048` | `(config)#` | 官方要求 ≥768 位，生产用 2048；同时启用 SSH server |
| 强制 SSHv2 | `ip ssh version 2` | `(config)#` | 不配则处于 v1/v2 兼容模式 |
| SSH 会话参数 | `ip ssh time-out 60` / `ip ssh authentication-retries 3` | `(config)#` | 超时秒数与认证重试次数 |
| 本地管理账号 | `username admin privilege 15 secret Cisco@123` | `(config)#` | 权限 15 为最高级 |
| 特权口令 | `enable secret Cisco@456` | `(config)#` | 进入 `#` 需此口令，务必与登录口令分开 |
| 进入 vty 线路 | `line vty 0 4` | `(config)#` | 行数随平台（9000 系列更多），只覆盖 0–4 |
| 只允许 SSH 登录 | `transport input ssh` | `(config-line)#` | 等于关闭 telnet 接入 |
| 本地认证 | `login local` | `(config-line)#` | 使用 `username` 配置的账号口令 |
| 空闲超时 | `exec-timeout 10 0` | `(config-line)#` | 10 分钟无操作自动断开，防占线 |
| 限制源地址登录 | `access-class MGMT in` | `(config-line)#` | ACL 未放行的源无法建立会话 |
| 查看 SSH 服务状态 | `show ip ssh` | 特权 EXEC `#` | 版本、认证重试、超时时间 |
| 查看在线 SSH 会话 | `show ssh` | 特权 EXEC `#` | 会话号、用户、加密算法，可 `disconnect` |
| 接口与 IP 摘要 | `show ip interface brief` | 特权 EXEC `#` | 最快确认端口 up/down 与地址 |
| 邻居与拓扑 | `show cdp neighbors detail` | 特权 EXEC `#` | 确认对端设备、端口与 IP（如已关闭 CDP 则不可用） |
| 错包统计 | `show interfaces counters errors` | 特权 EXEC `#` | CRC/align/runts，定位物理层问题 |
| 查看日志 | `show logging` | 特权 EXEC `#` | 含缓冲日志与日志服务器配置 |
| 实时看日志 | `terminal monitor` | 特权 EXEC `#` | 远程会话默认收不到日志，必须打开 |
| 连通性测试 | `ping 10.1.12.2` / `traceroute 10.1.12.2` | 特权 EXEC `#` | ping 测通断，traceroute 定位断点 |
| CPU 占用 | `show processes cpu sorted` | 特权 EXEC `#` | 排查控制面过载（如路由震荡、debug 未关） |
| 开/关调试 | `debug ip icmp` / `undebug all` | 特权 EXEC `#` | ⚠️ 必须成对使用，见高危节 |

## 典型示例

```text
SW1# configure terminal
SW1(config)# hostname SW1                             // 1. 先有主机名
SW1(config)# ip domain-name example.com               // 2. 再有域名
SW1(config)# crypto key generate rsa modulus 2048     // 3. 生成密钥，自动启用 SSH
SW1(config)# ip ssh version 2                         // 4. 只跑 SSHv2
SW1(config)# ip ssh time-out 60
SW1(config)# username admin privilege 15 secret Cisco@123
SW1(config)# enable secret Cisco@456
SW1(config)# line vty 0 4
SW1(config-line)# login local
SW1(config-line)# transport input ssh                 // 关闭 telnet
SW1(config-line)# exec-timeout 10 0
SW1(config-line)# end
SW1# show ip ssh
SW1# show ssh
SW1# copy running-config startup-config
```

排障常用组合（不改配置）：

```text
SW1# show ip interface brief
SW1# show cdp neighbors detail
SW1# show interfaces counters errors
SW1# terminal monitor
SW1# show logging | include LINK|DOWN                // 过滤链路抖动记录
SW1# ping 10.1.12.2 repeat 10 size 1500
SW1# traceroute 10.1.12.2
SW1# debug ip icmp                                  // 临时抓 ICMP，用完立刻关
SW1# undebug all
```

## 陷阱与版本差异

- 生成 RSA 密钥前必须已有 `hostname` **和** `ip domain-name`（官方步骤顺序），否则提示"请输入域名"；密钥与主机名绑定，改主机名后需 `crypto key zeroize rsa` 再重新生成。
- 官方限制：RSA 密钥长度必须 **≥768 位**；不配 `ip ssh version 2` 时 SSH 处于 v1/v2 兼容模式，SSHv1 从未标准化，生产应显式指定版本 2。
- `transport input ssh` 会同时挡掉 telnet：官方提示此时依赖 telnet 的功能（如 `session slot` 访问设备内模块）不可用；确有需要可写 `transport input ssh telnet`，但要接受明文风险。
- `line vty 0 4` 只覆盖前 5 条线路，Catalyst 9000 的 vty 数量更多，只改 0–4 时其余线路仍是默认策略；建议先 `show run | section line vty` 看清范围。
- `terminal monitor` 只对当前会话有效，重新登录后远程会话仍收不到日志；持久化要看 `show logging` 或配置 logging host。
- 密码恢复无法通过 SSH/网络完成，需 Console + ROMMON 流程（涉及重启与配置寄存器修改），属高危操作，务必双人复核。

## 高危操作 ⚠️

- ⚠️ 在未验证 SSH 可用的情况下执行 `transport input ssh`（关闭 telnet）：一旦密钥缺失或 ACL 写错，远程彻底失联，只能到现场用 Console 恢复；务必先另开会话验证 SSH 成功再关闭 telnet。
- ⚠️ `crypto key zeroize rsa`：删除 RSA 密钥会**自动关闭 SSH server**（官方明确），远程登录立即失效；仅在 Console 在场时执行，并准备好重新生成密钥。
- ⚠️ vty 上误配 `access-class` 或删除最后一个 `username`：管理面被锁死；变更前确认 ACL 放行管理网段，并保留 Console 通道。
- ⚠️ `debug all` 或忘记 `undebug all`：CPU 飙高、日志刷屏导致设备卡死甚至业务中断；调试限定到具体特性，并在同一会话内配 `undebug all` 定时收尾。

## 参考来源

1. [Secure Shell Version 2 Support（hostname/domain-name/RSA 密钥/SSHv2）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_ssh/configuration/15-mt/sec-usr-ssh-15-mt-book/sec-secure-shell-v2.html)
2. [SSH Terminal-Line Access（line/transport input ssh/login 认证）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/sec_usr_ssh/configuration/15-mt/sec-usr-ssh-15-mt-book/sec-ssh-term-line.html)
3. [System Management Configuration Guide, Cisco IOS XE 17.6.x（含 System Message Logs 章节）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg.html)
