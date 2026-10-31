---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: 远程管理与诊断
devices: ZXR10 5900/5950/6900/8900 系列交换机、ZXR10 2900E/1800/2800 系列路由器、5960/5928E 等
keywords: [中兴, ZXR10, ssh server enable, system-user, aaa-authentication-template, show username online, clear line vty, show processor, show logging alarm, copy ftp, startrun.dat, debug]
sources:
  - https://weril.me/zte/
  - https://developer.aliyun.com/article/1171627
  - https://www.tenable.com/audits/items/Tenable-ZTE_ROSNG-Best-Practice-v1.1.0.audit:5e2063deb095e2f25c1c74942dd69348
  - https://csl.zte.com.cn/manual_zh/topics/1684391333553.html
  - https://support.zte.com.cn/support/doccenter/DocumentProductHandBookDetail.aspx?sid=102&id=30768582&type=docfeedback
verified: medium
updated: 2026-10-30
---

# 中兴 · 远程管理与诊断

> ZXR10 的远程接入有两套并存写法：**老式 `username ... privilege`** 与 **ROSNG 的 `system-user` + `aaa-authentication-template` 模板绑定**，同型号不同版本可能只支持其中一套。
> 配置文件是 `flash:/cfg/startrun.dat`，保存靠 `write`；排障先看 `show username online`、`show processor`、`show logging alarm`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 开启 SSH 服务 | `ssh server enable` | 全局配置 `(config)#` | ROSNG 官方审计基线明确给出的写法；关闭用 `no ssh server enable` |
| 开启 FTP/SFTP 服务 | `ftp-server enable` | 全局配置 | 传版本/抓 dump 用；生产环境建议仅在维护窗口开启 |
| 老式建本地用户 | `username netadmin password Noc@2026 privilege 15` | 全局配置 | 权限 0~15，15 最高 |
| ROSNG 建用户 | `system-user` → `user-name admin` → `password <口令>` | 全局配置 → `system-user` 子视图 | 提示符随版本带名（如 `(config-sysuser)#`），以现场为准 |
| 绑定认证/授权模板 | `bind authentication-template 1` / `bind authorization-template 1` | `system-user` 子视图 | 两个模板必须都绑，否则登录后无权限 |
| 定义认证模板 | `aaa-authentication-template 2001` → `aaa-authentication-type local` | 全局配置 | `local` 即本地用户库认证 |
| 定义授权模板 | `aaa-authorization-template 2001` → `aaa-authorization-type none` | 全局配置 | 配合 `authorization-template` 定权限档位 |
| 模板定权限 | `authorization-template 1` → `local-privilege-level 15` | 全局配置 | 权限不够时表现为"只能 show 不能改" |
| 设置特权口令 | `enable secret level 15 0 <口令>` | 全局配置 | `0` 表示明文，`level 15` 对应最高档 |
| 管理口配地址 | `interface mgmt_eth` → `ip address 192.168.1.254 255.255.255.0` | 全局配置 → 接口视图 | 点分十进制掩码；带外管理首选 |
| 查看在线会话 | `show username online` | 特权视图 `ZXR10#` | 含用户名、协议、`Vty-No.`、来源 IP |
| 踢掉会话 | `clear line vty <n>` | 特权视图 | `n` 取 `Vty-No.`；带 `*` 的是自己，别清自己 |
| 查软件版本 | `show version software` | 特权视图 | 判断命令集差异的第一手依据 |
| 查运行配置 | `show running-config` / `show running-config all` | 特权/配置视图 | `all` 含默认值，核对"到底生效了什么" |
| 查 CPU 利用率 | `show processor` | 特权视图 | 各单板 CPU，debug 全开后第一时间看它 |
| 查告警日志 | `show logging alarm` | 特权视图 | 设备告警类信息；通用日志再配 `show logging` 核对 |
| 查接口状态 | `show interface gei_1/1` | 特权/配置视图 | 双工、速率、VLAN、收发统计 |
| 查三层接口 | `show ip interface brief` | 特权/配置视图 | 管理面通不通的第一步 |
| 查路由表 | `show ip route` / `show ip protocol routing` | 特权视图 | 部分平台区分协议表与转发表（`show ip forwarding route`） |
| 连通性测试 | `ping <ip>` / `traceroute <ip>` | 特权视图 | 支持 `ping vrf <vrf名> <ip>` 指定 VRF |
| 查 CPU 保护日志 | `show cpu-guard-log shelf 0 panel 1 level 1` | 特权视图 | cpu-guard 丢弃统计，判断"莫名丢包"是否被限速 |
| 文件系统 | `dir` / `cd <目录>` / `pwd` / `cp` / `delete` / `rename` | 特权视图 | `flash:/cfg` 存 `startrun.dat`，`flash:/img` 存版本 |
| 保存配置 | `write` | 特权视图 | 写入 `flash:/cfg/startrun.dat`，回显 `[ok]` |
| 异机备份 | `copy root /cfg/startrun.dat ftp://<ip>/startrun.dat@<user>:<pwd>` | 特权视图 | 四段式；反向写即恢复 |
| 上传文件（另一写法） | `copy ftp //<ip>/<文件名>@<user>:<pwd> root: <文件名>` | 特权视图 | 与上一条**参数顺序相反**，按版本选 |
| 重启 | `reload` | 特权视图 | 各版本也可能是 `reboot`，以官方手册确认 |

## 典型示例

```text
ZXR10>enable                                        // 用户视图 → 特权视图
Password:
ZXR10#configure terminal
ZXR10(config)#enable secret level 15 0 Adm@2026     // 最高档特权口令
// 写法 A：老式本地用户（多数 5900/6900 版本可用）
ZXR10(config)#username netadmin password Noc@2026 privilege 15
// 写法 B：ROSNG 模板绑定（子视图提示符随版本带名，以现场为准）
ZXR10(config)#aaa-authentication-template 2001
ZXR10(config)#aaa-authentication-type local
ZXR10(config)#aaa-authorization-template 2001
ZXR10(config)#aaa-authorization-type none
ZXR10(config)#authentication-template 1
ZXR10(config)#bind aaa-authentication-template 2001
ZXR10(config)#authorization-template 1
ZXR10(config)#bind aaa-authorization-template 2001
ZXR10(config)#local-privilege-level 15
ZXR10(config)#system-user
ZXR10(config)#user-name admin
ZXR10(config)#password Noc@2026
ZXR10(config)#bind authentication-template 1
ZXR10(config)#bind authorization-template 1
ZXR10(config)#exit
ZXR10(config)#ssh server enable                     // 官方审计基线写法
ZXR10(config)#interface mgmt_eth                    // 带外管理口
ZXR10(config-if)#ip address 192.168.1.254 255.255.255.0
ZXR10(config-if)#exit
ZXR10(config)#end
ZXR10#write                                         // 保存到 flash:/cfg/startrun.dat
ZXR10#show version software
ZXR10#show username online                          // 确认新账号与在线会话
```

排障与备份：

```text
ZXR10#show ip interface brief                       // 管理 IP 是否 up
ZXR10#show interface gei_1/24                       // 上联口双工/速率/CRC
ZXR10#show ip route                                 // 去程路由是否在表
ZXR10#ping 192.168.1.1                              // 逐跳定位，先近后远
ZXR10#traceroute 10.0.0.1
ZXR10#show processor                                // CPU 是否异常
ZXR10#show logging alarm                            // 告警时间线
ZXR10#show cpu-guard-log shelf 0 panel 1 level 1    // 是否被 cpu-guard 限速
ZXR10#dir                                           // 确认 startrun.dat 大小/时间
ZXR10#copy root /cfg/startrun.dat ftp://10.0.0.9/startrun.dat@backup:Bak@2026
// 恢复：把文件反向拷回，再 reload 生效
ZXR10#copy ftp://10.0.0.9/startrun.dat@backup:Bak@2026 root /cfg/startrun.dat
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：ZXR10 平台跨度极大（2900E/5928E/5960/5900/6900/8900 与 1800 路由器），`ssh server enable`、`system-user`、`ip route` 视图名、`copy` 参数顺序都随版本变化，落地前请用 `?` 与 `show version software` 核对，或让售后按版本出配置样例。
- 本地用户有**两套并存写法**：老式 `username <名> password <口令> privilege <0-15>` 与 ROSNG 的 `system-user` + `aaa-authentication-template`/`authorization-template` 模板绑定。同一台设备上只认其中一套，抄错表现为"用户建了但登录失败"或"能登录但改不了配置"。
- 模板绑定必须**认证与授权成对**：只 `bind authentication-template` 不绑 `authorization-template`，能过认证却没有权限档位，看起来像"密码对了但没权限"。
- `copy` 有两种参数顺序：`copy root /cfg/startrun.dat ftp://...` 与 `copy ftp //<ip>/<文件>@<user>:<pwd> root: <文件>`。把备份方向的写法直接反过来用，容易报 `%Error 29: Invalid directory or filename` 之类的错。
- 接口名在不同平台不一致：盒式交换机常见 `gei_1/1`，部分框式平台是 `gei-0/1/1/1` 三段式；脚本批量下发前先 `show interface` 核对实际命名。
- 排障命令分层：`show ip route` 看路由表，部分平台另有 `show ip protocol routing`（协议表）与 `show ip forwarding route`（转发表）；"路由表有但业务不通"时要两张表对照。
- `show running-config` 默认不显示默认值，某些功能（如 `cpu-guard`）连默认配置都隐藏；要核对默认值用 `show running-config all` 或对应模块的 `show ... all`。

## 高危操作 ⚠️

- ⚠️ `no ssh server enable`（或关闭 telnet/ftp 服务）会当场切断远程登录入口，若此时无 Console/带外通道即彻底失联；改前先确认 `show username online` 有其他会话、并确保 `interface mgmt_eth` 可用。
- ⚠️ `clear line vty <n>` 踢错会话：`n` 索引与 `show username online` 的 `Vty-No.` 必须逐个对齐，误清他人正在变更的会话会中断其配置下发；带 `*` 的行是自己，清自己直接断线。
- ⚠️ `debug` 类命令全开（尤其 packet/协议级 debug）会迅速拉高 CPU 并刷满日志缓冲，导致协议邻居震荡甚至业务中断。纪律：**只在维护窗口、最小范围（单模块/单接口/单 IP）开启，随时备好 `undebug all`（部分版本为 `no debug all`），并同步 `show processor` 盯 CPU**；具体 debug 命令名按版本以官方手册为准。
- ⚠️ 修改 `enable secret` / `privilege` / `local-privilege-level` 或删除唯一管理员账号，可能把自己锁在设备外；改前保留 Console，并先建好一个新的 15 级账号验证可用。
- ⚠️ `write` 是覆盖式保存：错误配置一旦落盘，回退只能靠事先导出的 `startrun.dat`。变更前固定动作「`show running-config` + `copy` 导出到 FTP」。
- ⚠️ `delete` 掉 `flash:/cfg/startrun.dat` 等于清空启动配置，重启后设备回初始状态、业务全断；`reload`/`reboot` 前必须确认配置已 `write` 且带外通道可用。

## 参考来源

1. [ZXR10 5928E-FI / 5960-32DL 实战配置记录（`ssh server enable`、`ftp-server enable`、`system-user` + 模板绑定、`copy tftp/ftp`、`cpu-guard`、`show cpu-guard-log`）（weril.me）](https://weril.me/zte/)
2. [中兴设备 show 命令大全（`show processor`、`show logging alarm`、`show ip protocol routing`、`ping vrf`）（阿里云开发者社区）](https://developer.aliyun.com/article/1171627)
3. [Tenable ZTE ROSNG Best Practices · 1.1 Secure Login and telnet Disabling（明确 `ZXR10(config)#ssh server enable`）](https://www.tenable.com/audits/items/Tenable-ZTE_ROSNG-Best-Practice-v1.1.0.audit:5e2063deb095e2f25c1c74942dd69348)
4. [中兴官方手册站 · CLI 配置（`write` 保存与 startrun.dat 导出）](https://csl.zte.com.cn/manual_zh/topics/1684391333553.html)（官方）
5. [中兴技术支持文档中心（Tenable 条目引用的官方手册入口，供按版本核对命令）](https://support.zte.com.cn/support/doccenter/DocumentProductHandBookDetail.aspx?sid=102&id=30768582&type=docfeedback)（官方）
