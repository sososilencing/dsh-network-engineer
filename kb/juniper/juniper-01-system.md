---
vendor: 瞻博（Juniper）
os: Junos
topic: 系统管理与视图
devices: EX2300/EX4300 系列交换机、MX 系列路由器、SRX 系列防火墙等 Junos 设备
keywords: [候选配置, configure, commit, commit confirmed, rollback, show | compare, display set, set system login, NTP, 恢复出厂]
sources:
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html
  - https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/getting-started.html
verified: high
updated: 2026-10-30
---

# 瞻博（Juniper）· 系统管理与视图

> Junos 是「候选配置 + 提交」模型：`set`/`delete` 只改候选库，必须 `commit` 才生效；远程改设备必须用 `commit confirmed N` 兜底。与 Cisco/华为「回车即生效、再 save/write 落盘」的模型完全不同，这是迁移到 Junos 最容易踩的坑。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入配置模式 | `configure` | 操作模式 `user@host>` | 提示符变 `#`，出现 `[edit]` 层级横幅 |
| 退出配置模式 | `exit` | `[edit]` | 有未提交改动会提示；`exit configuration-mode` 免提示退出 |
| 设定语句（不改层级） | `set system host-name CORE-EX-01` | `[edit]` | `set` 是 Junos 最常用的写法 |
| 进入层级（改层级） | `edit interfaces ge-0/0/1` | `[edit]` | 只移动光标层级，不提交 |
| 删除语句 | `delete interfaces ge-0/0/1` | `[edit]` | 删除整棵子树，⚠️ 见高危节 |
| 挂起/恢复语句 | `deactivate ...` / `activate ...` | `[edit]` | 保留配置但不生效，等价注释 |
| 查看候选配置 | `show` | `[edit]` | 花括号层级格式 |
| 查看 set 格式 | `show \| display set` | `[edit]` | 直接输出可复制的 set 命令，最适合备份/贴单 |
| 与已提交配置比对 | `show \| compare` | `[edit]` | 只显示本次会话的 +/− 差异 |
| 提交 | `commit` | `[edit]` | 提交全部候选配置并生效 |
| 带确认提交 | `commit confirmed 5` | `[edit]` | 5 分钟内未再 commit 自动回滚（默认 10 分钟） |
| 提交并退出 | `commit and-quit` | `[edit]` | 等价 `commit` + `exit` |
| 仅校验语法 | `commit check` | `[edit]` | 不生效，返回 commit check completed successfully |
| 回滚候选配置 | `rollback` | `[edit]` | 恢复到最近一次已提交配置（仍需 commit 生效） |
| 回滚到历史版本 | `rollback 1` | `[edit]` | 0 为当前提交，1 为上一次提交，依次递增 |
| 查看提交历史 | `show system commit` | 操作模式 | 含 rollback 序号、时间、提交人 |
| 配置模式内执行操作命令 | `run show interfaces terse` | `[edit]` | 所有操作命令加 `run` 前缀 |
| 备份配置到文件 | `save /var/tmp/EX01-20261030.conf` | `[edit]` | 配合 `file copy` 取回本地 |
| 新建管理账号 | `set system login user ops class super-user authentication plain-text-password` | `[edit]` | 提交时提示输入两次明文密码 |
| 配置 NTP | `set system ntp server 10.0.0.100` | `[edit]` | — |
| 配置时区 | `set system time-zone Asia/Shanghai` | `[edit]` | 只改时区，时间靠 NTP 同步 |

## 典型示例

```text
user@EX-01> configure                                  // 进入配置模式，PFE 不变，仅候选库
[edit]
user@EX-01# set system host-name CORE-EX-01
user@EX-01# set system time-zone Asia/Shanghai
user@EX-01# set system ntp server 10.0.0.100
user@EX-01# set system login user ops class super-user authentication plain-text-password
New password:                                          // 交互输入两次
Retype new password:
user@EX-01# show | compare                             // 提交前先看差异
[edit system]
+  host-name CORE-EX-01;
+  time-zone Asia/Shanghai;
user@EX-01# commit check                               // 只校验语法
configuration check succeeds
user@EX-01# commit confirmed 5                         // 远程改设备必用：5 分钟不回确认即自动回滚
commit confirmed will be automatically rolled back in 5 minutes unless confirmed
commit complete
user@EX-01# run show system commit                     // 确认本次提交已记录
user@EX-01# commit                                     // 5 分钟内二次 commit = 确认，取消定时回滚
commit complete
user@EX-01# save /var/tmp/EX01-20261030.conf           // 备份当前候选配置（文本）
user@EX-01# exit
Exiting configuration mode
user@EX-01> show configuration | display set | save /var/tmp/EX01-set.txt   // 导出 set 清单
```

## 陷阱与版本差异

- `commit confirmed` 默认等待 **10 分钟**，命令里给数字才会改时长；超时未再 `commit` 会**整体回滚到提交前**，同时给所有在线用户广播提示。
- `rollback` 只是把历史版本加载进候选库，**不 commit 不生效**；`rollback 0` 用于丢弃本会话未提交的改动。
- 回滚版本数量说法有差异：CLI 指南写「保留最近 10 个已提交配置可用于 rollback」，提交机制文档写「系统最多保存 49 个 rollback 文件（juniper.conf.1.gz ~ 49）」，两者以官方手册对应版本为准。
- `commit` 是**全量提交**：多人同时编辑共享候选库时，任何人 commit 都会把其他人未提交的改动一起提交；要隔离请用 `configure private`。
- ELS 与非 ELS 平台的二层语法不同（`interface-mode` 对 `port-mode`），EX 新出厂版本基本为 ELS；跨版本照抄命令前先 `show version` 确认。
- root 账号默认无密码且**不允许 Telnet 登录**；要允许 root 走 SSH 必须显式 `set system services ssh root-login allow`（默认 deny）。
- `set` 只解决「叶子语句」，删除单个成员要用 `delete` 或 `delete ... [ 成员 ]`；忘记 `commit` 直接 `exit` 会提示且改动丢失。

## 高危操作 ⚠️

- ⚠️ `commit`：**直接生效且无自动回滚**。改远程可达性（管理口、SSH、路由/ACL）时必须改用 `commit confirmed N`，否则一旦配错即失联，只能到现场。
- ⚠️ `delete` 整个层级（如 `delete interfaces`、`delete protocols ospf`）：连带删除所有子语句，业务中断且候选库立即变化；先 `show | compare` 确认范围，再考虑 `deactivate` 代替删除。
- ⚠️ `request system zeroize`：清空全部配置与密钥并重启，设备回到出厂状态，重启后管理口不可达，只能 Console 以 root 登录（SRX 带 VM Host 的机型用 `request vmhost zeroize`）。
- ⚠️ `request system reboot`：中断业务；重启前先 `commit` 并 `save`/`request system snapshot` 备份，且在维护窗口执行。

## 参考来源

1. [Commit the Configuration（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/junos-configuration-commit.html)
2. [CLI Configuration Mode Overview（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/cli-configuration.html)
3. [Getting Started: A Quick Tour of the CLI（juniper.net）](https://www.juniper.net/documentation/us/en/software/junos/cli/topics/topic-map/getting-started.html)
