---
vendor: 思科（Cisco）
os: IOS / IOS-XE
topic: 系统管理与视图
devices: Catalyst 9200/9300、ISR 1000/4000、ASR 1000 等
keywords: [IOS 视图, enable, configure terminal, copy running-config startup-config, write memory, configure replace, NTP, ntp server, username privilege 15, banner motd]
sources:
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg/managing_configuration_files.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg/administering_the_device.html
  - https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg.html
verified: high
updated: 2026-10-30
---

# 思科 · 系统管理与视图

> 适用于 IOS/IOS-XE 的 Catalyst 交换机与 ISR/ASR 路由器：视图层级、配置保存与回滚、时钟与 NTP、管理账号。**关键差异**：IOS 配置即时生效但**不会自动落盘**，必须显式 `copy running-config startup-config`（或 `write memory`）才能重启不丢。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入特权 EXEC | `enable` | 用户 EXEC `<Switch>` | 提示符 `>` 变 `#`，需 enable secret/password |
| 进入全局配置 | `configure terminal` | 特权 EXEC `#` | 提示符变 `(config)#`，可简写 `conf t` |
| 直回特权 EXEC | `end` | 任意配置视图 | 等价 Ctrl+Z；`exit` 只退一层 |
| 修改设备名 | `hostname CORE-SW-01` | `(config)#` | 提示符即时更新；生成 SSH 密钥前必须先设 |
| 查看运行配置 | `show running-config` | 特权 EXEC `#` | 内存中的当前配置，`show run` 简写 |
| 保存配置 | `copy running-config startup-config` | 特权 EXEC `#` | 官方推荐的保存方式，写入 NVRAM |
| 保存配置（旧写法） | `write memory` | 特权 EXEC `#` | 与上一条等效，`wr` 简写，老习惯用法 |
| 查看启动配置 | `show startup-config` | 特权 EXEC `#` | 与 running 对比可发现"改了没存" |
| 整份配置替换/回滚 | `configure replace flash:backup.cfg` | 特权 EXEC `#` | 全量替换而非合并，可配 `revert timer` 自动回退 |
| 查看版本与运行时长 | `show version` | 特权 EXEC `#` | 机型、IOS-XE 版本、uptime、上次重启原因 |
| 设置时区 | `clock timezone CST 8` | `(config)#` | 只定义偏移，不给具体时刻 |
| 手工对时 | `clock set 14:30:00 30 Oct 2026` | 特权 EXEC `#` | 参数顺序随版本，以官方手册为准 |
| 配置 NTP 服务器 | `ntp server 10.0.0.100` | `(config)#` | 客户端模式；`ntp peer` 为对称主动模式 |
| 设为权威时钟源 | `ntp master 3` | `(config)#` | 低 stratum 会覆盖有效时间源，慎用 |
| 限制 NTP 访问 | `ntp access-group serve-only 10` | `(config)#` | 基于 ACL 的 NTP 访问控制 |
| 创建管理账号 | `username admin privilege 15 secret Cisco@123` | `(config)#` | `secret` 存加密口令，优于 `password` |
| 进入 console 线路 | `line console 0` | `(config)#` | 提示符变 `(config-line)#`；`line vty 0 4` 见 08 篇 |
| 登录横幅 | `banner motd #Authorized access only#` | `(config)#` | 分隔符可自选，登录前显示 |
| 清空启动配置 | `write erase` | 特权 EXEC `#` | ⚠️ 见高危节 |

## 典型示例

```text
Switch> enable
Switch# configure terminal
Switch(config)# hostname CORE-SW-01                  // 主机名，SSH 生成密钥前必配
CORE-SW-01(config)# clock timezone CST 8             // UTC+8
CORE-SW-01(config)# ntp server 10.0.0.100            // NTP 优先级高于手工对时
CORE-SW-01(config)# ntp access-group serve-only 10   // 只允许 ACL 10 内主机来同步
CORE-SW-01(config)# username admin privilege 15 secret Cisco@123   // 本地最高权限账号
CORE-SW-01(config)# banner motd #Authorized access only#
CORE-SW-01(config)# end
CORE-SW-01# show clock                               // 确认时间源与偏差状态
CORE-SW-01# show run | include ntp                   // 核对 NTP 是否已下发
CORE-SW-01# copy running-config startup-config       // 落盘，否则重启丢失
CORE-SW-01# show startup-config | include hostname   // 核对已保存内容
```

变更前备份 → 变更 → 回滚（推荐流程）：

```text
CORE-SW-01# copy running-config flash:backup-20261030.cfg            // 变更前备份
CORE-SW-01# configure replace flash:backup-20261030.cfg              // 整份替换
CORE-SW-01# configure replace flash:backup-20261030.cfg revert timer 5   // 5 分钟不确认则自动回退
```

## 陷阱与版本差异

- IOS 配置**不会自动保存**：只有执行 `copy running-config startup-config` / `write memory` 后才写入启动配置，未保存的改动重启即丢；这点与 Junos `commit`、EOS 显式保存一样，都需要人为落盘。
- Catalyst 9000 的 VLAN 1–1005 保存在 `vlan.dat`，`write erase` 不清该文件；官方 VLAN 章节明确要求重置设备时连同 `vlan.dat` 一起删除再重启，否则残留 VLAN 配置。
- `configure replace` 是**整份替换**，目标文件里没写的接口/VLAN/路由会被删除；`copy tftp: running-config` 是**合并**语义，两者不能混用。
- `clock timezone` 只设时区不产生正确时刻，必须靠 `ntp server` 或 `clock set`；日志时间戳错乱、证书校验失败多与此相关。
- 本地口令 `secret` 与 `password`（type 7/可逆）存储方式不同，新版本一律用 `secret`；关键字支持随 IOS/IOS-XE 版本演进，个别参数以官方手册为准。

## 高危操作 ⚠️

- ⚠️ `write erase`（`erase startup-config`）：清空启动配置，重启后设备无配置、业务与远程管理全失联；执行前必须 `copy running-config flash:xxx.cfg` 或 `copy running-config tftp:` 导出，并在维护窗口操作。
- ⚠️ `reload`：整机重启，业务中断数分钟；重启前确认已保存配置、`boot system` 指向可用镜像，并避开业务高峰。
- ⚠️ `configure replace flash:xxx.cfg`：整份替换运行配置，缺失项会被直接删除；务必先备份当前配置，或用 `revert timer` 留出自动回退窗口。
- ⚠️ Console/vty 上改错 AAA 或误删本地账号会导致无法登录，只能进 ROMMON 恢复；变更期间保留另一条可用登录通道（SSH + Console 双通道）。

## 参考来源

1. [Managing Configuration Files, System Management Configuration Guide 17.6.x（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg/managing_configuration_files.html)
2. [Administering the Device（时钟/NTP/主机名/横幅）（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg/administering_the_device.html)
3. [System Management Configuration Guide, Cisco IOS XE 17.6.x (Catalyst 9300)（cisco.com）](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-6/configuration_guide/sys_mgmt/b_176_sys_mgmt_9300_cg.html)
