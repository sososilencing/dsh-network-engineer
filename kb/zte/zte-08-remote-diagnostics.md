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
verified: medium
updated: 2026-10-30
---

# 中兴 · 远程管理与诊断

> ZXR10 远程接入有两套并存写法：**老式 `username ... privilege`** 与 **ROSNG 的 `system-user` + AAA 模板绑定**，同型号不同版本可能只支持其中一套；配置文件是 `flash:/cfg/startrun.dat`，保存靠 `write`。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 开启 SSH 服务 | `ssh server enable` | 全局配置 `(config)#` | ROSNG 官方审计基线写法；关闭用 `no ssh server enable` |
| 老式建本地用户 | `username netadmin password Noc@2026 privilege 15` | 全局配置 | 权限 0~15，15 最高 |
| ROSNG 建用户 | `system-user` → `user-name admin` → `password <口令>` | 全局配置 → `system-user` 子视图 | 子视图提示符随版本带名，以现场为准 |
| ROSNG 绑模板 | `bind authentication-template 1` / `bind authorization-template 1` | `system-user` 子视图 | 两模板必须成对绑定，漏绑则无权限档位 |
| AAA 模板链 | `aaa-authentication-template 2001` → `aaa-authentication-type local`；`authorization-template 1` → `local-privilege-level 15` | 全局配置 | `local` 即本地用户库认证 |
| 设置特权口令 | `enable secret level 15 0 <口令>` | 全局配置 | `0` 为明文，`level 15` 最高档 |
| 管理口配地址 | `interface mgmt_eth` → `ip address 192.168.1.254 255.255.255.0` | 全局配置 → 接口视图 | 点分十进制掩码；带外管理首选 |
| 查看在线会话 | `show username online` | 特权视图 `ZXR10#` | 含用户名、协议、`Vty-No.`、来源 IP |
| 踢掉会话 | `clear line vty <n>` | 特权视图 | `n` 取 `Vty-No.`；带 `*` 的是自己，别清自己 |
| 查软件版本 | `show version software` | 特权视图 | 判断命令集差异的第一手依据 |
| 查运行配置 | `show running-config` / `show running-config all` | 特权/配置视图 | `all` 含默认值，核对"到底生效了什么" |
| 查 CPU 利用率 | `show processor` | 特权视图 | 各单板 CPU，debug 全开后第一时间看它 |
| 查告警/日志 | `show logging alarm` | 特权视图 | 告警类信息；通用日志再配 `show logging` 核对 |
| 查接口与三层 | `show interface gei_1/1` / `show ip interface brief` | 特权/配置视图 | 前者看双工/速率/CRC，后者看管理面通不通 |
| 查路由表 | `show ip route` / `show ip protocol routing` | 特权视图 | 部分平台区分协议表与转发表（`show ip forwarding route`） |
| 连通性测试 | `ping <ip>` / `traceroute <ip>` | 特权视图 | 支持 `ping vrf <vrf名> <ip>` 指定 VRF |
| 文件系统 | `dir` / `cd <目录>` / `pwd` / `cp` / `delete` / `rename` | 特权视图 | `flash:/cfg` 存 `startrun.dat`，`flash:/img` 存版本 |
| 保存与备份 | `write`；`copy root /cfg/startrun.dat ftp://<ip>/startrun.dat@<user>:<pwd>` | 特权视图 | `write` 写 `startrun.dat`；`copy` 为四段式 |
| 备份（另一写法） | `copy ftp //<ip>/<文件名>@<user>:<pwd> root: <文件名>` | 特权视图 | 与上一条**参数顺序相反**，按版本选 |

## 典型示例

```text
ZXR10>enable
Password:
ZXR10#configure terminal
ZXR10(config)#username netadmin password Noc@2026 privilege 15   // 写法 A：老式本地用户
ZXR10(config)#ssh server enable
ZXR10(config)#interface mgmt_eth
ZXR10(config-if)#ip address 192.168.1.254 255.255.255.0
ZXR10(config-if)#exit
ZXR10(config)#end
ZXR10#write                                     // 保存到 flash:/cfg/startrun.dat
ZXR10#show username online                      // 确认新账号与在线会话
```

```text
// 写法 B：ROSNG 模板绑定（子视图提示符随版本带名，以现场为准）
ZXR10(config)#aaa-authentication-template 2001
ZXR10(config)#aaa-authentication-type local
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
```

```text
ZXR10#show ip interface brief                   // 管理 IP 是否 up
ZXR10#show ip route                             // 去程路由是否在表
ZXR10#ping 192.168.1.1                          // 逐跳定位，先近后远
ZXR10#show processor                            // CPU 是否异常
ZXR10#show logging alarm                        // 告警时间线
ZXR10#dir                                       // 确认 startrun.dat 大小/时间
ZXR10#copy root /cfg/startrun.dat ftp://10.0.0.9/startrun.dat@backup:Bak@2026
ZXR10#copy ftp://10.0.0.9/startrun.dat@backup:Bak@2026 root /cfg/startrun.dat   // 恢复方向
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：ZXR10 平台跨度极大（2900E/5928E/5960/5900/6900/8900 与 1800 路由器），`ssh server enable`、`system-user`、`copy` 参数顺序、视图名都随版本变化，落地前用 `?` 与 `show version software` 核对，或请售后按版本出配置样例。
- 本地用户有**两套并存写法**：老式 `username <名> password <口令> privilege <0-15>` 与 ROSNG 的 `system-user` + AAA 模板绑定。设备上只认其中一套，抄错表现为"用户建了却登录失败"或"能登录但改不了配置"。
- 模板绑定必须**认证与授权成对**：只 `bind authentication-template` 不绑 `authorization-template`，能过认证却没有权限档位，看起来像"密码对了但没权限"。
- `copy` 有两种参数顺序：`copy root /cfg/startrun.dat ftp://...` 与 `copy ftp //<ip>/<文件>@<user>:<pwd> root: <文件>`；把备份方向直接反过来用易报 `%Error 29: Invalid directory or filename` 一类错误。
- 接口名跨平台不一致：盒式常见 `gei_1/1`，部分框式平台是 `gei-0/1/1/1` 三段式；脚本批量下发前先 `show interface` 核对实际命名。
- `show running-config` 默认不显示默认值，某些功能（如 `cpu-guard`）连默认配置都隐藏；核对默认值要用 `show running-config all`。

## 高危操作 ⚠️

- ⚠️ `no ssh server enable`（或关闭 telnet/ftp 服务）会当场切断远程登录入口，若此时无 Console/带外通道即彻底失联；改前确认 `show username online` 有其他会话且 `interface mgmt_eth` 可用。
- ⚠️ `clear line vty <n>` 踢错会话：`n` 必须与 `show username online` 的 `Vty-No.` 逐个对齐，误清他人正在变更的会话会中断其配置下发；带 `*` 的行是自己，清自己直接断线。
- ⚠️ `debug` 类命令全开（尤其 packet/协议级）会迅速拉高 CPU、刷满日志缓冲，导致邻居震荡甚至业务中断。纪律：**只在维护窗口、最小范围（单模块/单接口/单 IP）开启，随时备好 `undebug all`（部分版本为 `no debug all`），同步 `show processor` 盯 CPU**；具体 debug 命令名以官方手册为准。
- ⚠️ 修改 `enable secret` / `privilege` / `local-privilege-level` 或删除唯一管理员账号，可能把自己锁在设备外；改前保留 Console 并先建好一个新的 15 级账号验证。
- ⚠️ `write` 是覆盖式保存：错误配置一旦落盘只能靠事先导出的 `startrun.dat` 回退。变更前固定动作「`show running-config` + `copy` 导出到 FTP」。
- ⚠️ `delete` 掉 `flash:/cfg/startrun.dat` 等于清空启动配置，重启回初始状态、业务全断；`reload`/`reboot` 前必须确认已 `write` 且带外通道可用。

## 参考来源

1. [ZXR10 5928E-FI / 5960-32DL 实战配置记录（`ssh server enable`、`ftp-server enable`、`system-user` + 模板绑定、`copy tftp/ftp`、`cpu-guard`）（weril.me）](https://weril.me/zte/)
2. [中兴设备 show 命令大全（`show processor`、`show logging alarm`、`show ip protocol routing`、`ping vrf`）（阿里云开发者社区）](https://developer.aliyun.com/article/1171627)
3. [Tenable ZTE ROSNG Best Practices · 1.1 Secure Login and telnet Disabling（明确 `ZXR10(config)#ssh server enable`）](https://www.tenable.com/audits/items/Tenable-ZTE_ROSNG-Best-Practice-v1.1.0.audit:5e2063deb095e2f25c1c74942dd69348)
4. [中兴官方手册站 · CLI 配置（`write` 保存与 startrun.dat 导出）](https://csl.zte.com.cn/manual_zh/topics/1684391333553.html)（官方）
