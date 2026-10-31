---
vendor: 中兴（ZTE）
os: ZXR10 ROSNG / ZROS
topic: 系统管理与视图
devices: ZXR10 5900/5950/6900/8900 系列交换机、ZXR10 1800/2800 系列路由器等
keywords: [中兴, ZTE, ZXR10, configure terminal, write, startrun.dat, hostname, enable secret, privilege, multi-user configure, show running-config]
sources:
  - https://csl.zte.com.cn/manual_zh/topics/1684391333553.html
  - https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html
  - https://www.cnblogs.com/ys961206/p/17690916.html
  - http://szdingwei.net/html/6432195423.html
verified: medium
updated: 2026-10-30
---

# 中兴 · 系统管理与视图

> ZXR10 的 CLI 是「Cisco 风格 + 自有细节」：`enable` → `configure terminal` → `write` 三件套与 Cisco 一致，
> 但接口名是 `gei_1/1` 这种「类型_槽位/端口」写法，配置文件叫 `startrun.dat`，且**多数版本默认只允许一个终端同时进入配置模式**。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入特权模式 | `enable` | 用户视图 `ZXR10>` | 键入特权口令后提示符变 `ZXR10#`；提示符本身不区分 hostname 与模式，靠有无 `#`/`(config)` 判断 |
| 返回用户视图 | `disable` | 特权视图 `ZXR10#` | 降回用户视图；`exit` 是逐级退出当前视图 |
| 进入全局配置 | `configure terminal`（可缩写 `con t`） | 特权视图 | 提示符变 `ZXR10(config)#`，屏幕提示 `End with CTRL/Z`（`CTRL/Z` 或 `end` 直接回特权视图） |
| 设备命名 | `hostname zte-core-01` | 全局配置 `(config)#` | 命名后提示符立即变为 `zte-core-01(config)#`；与 Cisco `hostname` 同义 |
| 保存配置 | `write` | 特权视图 `ZXR10#` | 等价 Cisco `copy run start`，回显 `Building configuration..... [ok]`；把 running 写入 `flash:/cfg/startrun.dat` |
| 查看运行配置 | `show running-config`（`show run`） | 特权/配置视图 | ZXR10 允许在配置视图直接执行 `show` 类命令，不必退回特权模式 |
| 查看单接口配置 | `show running-config interface gei_2/4` | 特权/配置视图 | 只输出该接口配置，排障时比整篇 `show run` 快 |
| 查看软件版本 | `show version software` | 特权视图 | 输出含 `ZXR10 ROS Version ...`，是查版本差异的第一手依据 |
| 查看硬件/单板 | `show version hardware` | 特权视图 | 含主控与各线卡型号、内存、序列号 |
| 设置系统时间 | `clock set 23:12:01 feb 23 2026` | 特权视图 | 时间错会影响日志与告警时间戳，改时间前先与网管对时 |
| 登录问候语 | `banner incoming #You have entered a secured system.#` | 全局配置 | 以 `#` 为分隔符，分隔符可自定 |
| 设置特权口令 | `enable secret zte@2026` | 全局配置 | 可带等级：`enable secret level 8 <口令>`，让不同权限档位用不同口令 |
| 创建本地用户 | `username who password who [privilege 0-15]` | 全局配置 | 权限 0~15，0 最低 15 最高；不写 `privilege` 时为该版本默认级别 |
| 查看当前权限 | `show privilege` | 特权视图 | 输出 `Current privilege level is 15`，判断"命令敲不进去"是否权限档位问题 |
| 提权/降权 | `enable <level>` / `disable <level>` | 特权视图 | 按等级提权需输入对应 `enable secret level` 口令 |
| 查看在线用户 | `show username online` | 特权视图 | 列出用户名、协议、`Vty-No.`、来源 IP、登录时间，可定位占用配置锁的会话 |
| 踢掉在线会话 | `clear line vty <n>` | 特权视图 | `n` 取 `show username online` 的 `Vty-No.`；带 `*` 的是自己，不要清自己 |
| 允许多人同时配置 | `multi-user configure` | 全局配置 | 默认关闭；不开时第二个人进 config 会报 `%Error 140357: Simultaneous configs not allowed` |
| 配置文件系统 | `dir` / `cd <目录>` / `pwd` | 特权视图 | `flash:/cfg` 存配置（`STARTRUN.DAT`）、`flash:/img` 存版本（`zxr10.zar`）、`flash:/data` 存异常信息 |
| 配置导出/导入 | `copy root /cfg/startrun.dat ftp://<ip>/startrun.dat@<user>:<pwd>` | 特权视图 | 四段式 `copy <源设备> <源文件> <目的设备> <目的文件>`；反向写即恢复 |

## 典型示例

```text
ZXR10>enable                                  // 用户视图 → 特权视图，需输入特权口令
Password:
ZXR10#configure terminal                      // 进入全局配置，提示 End with CTRL/Z
ZXR10(config)#hostname core-sw-01             // 命名，提示符随之变化
core-sw-01(config)#multi-user configure       // 允许其他终端同时进配置模式（默认不允许）
core-sw-01(config)#enable secret level 8 Adm@2026
core-sw-01(config)#username netadmin password Noc@2026 privilege 15
core-sw-01(config)#banner incoming #Authorized access only#
core-sw-01(config)#end                        // 等价 CTRL/Z，回特权视图
core-sw-01#clock set 23:12:01 feb 23 2026
core-sw-01#show running-config                // 核对配置
core-sw-01#write                              // 保存到 flash:/cfg/startrun.dat
Building configuration.....
[ok]
core-sw-01#copy root /cfg/startrun.dat ftp://10.0.0.9/startrun.dat@backup:Bak@2026   // 异机备份
core-sw-01#show username online                // 排障：谁占着配置锁
core-sw-01#clear line vty 2                    // 踢掉占用会话后重新 con t
```

## 陷阱与版本差异

- 本机型/版本命令以官方手册为准：ZXR10 平台跨度极大（1800 老路由器、5900/6900/8900 交换机版本差异明显），凡本文未直接标注回显的，务必先用 `?` 与 `show version software` 核对。
- 视图层级与 Cisco 相同但提示符不同：全局是 `(config)#`、接口是 `(config-if)#`、VLAN 是 `(config-vlan)#`、路由协议是 `(config-router)#`；接口视图在部分平台/版本还会显示成 `(config-fei_1/1)#`、`(config-if-vlan10)#` 这类带名的形式，别当成进入了另一台设备。
- 保存命令是 `write`，不是 `copy running-config startup-config`（后者在 ZXR10 上不保证可用）；部分较新版本另提供 `save`，两者是否都支持以版本为准。
- 配置文件不是 `startup-config`：启动配置是 `flash:/cfg/startrun.dat`（课件中写作 `STARTRUN.DAT`），排障时用它判断"是否真的保存过"。
- 默认只允许单个终端进入配置模式。第二个人 `con t` 会报 `%Error 140357: Simultaneous configs not allowed. Locked from vty0 (x.x.x.x)`，需 `multi-user configure` 或清掉占用 vty，这不是权限配错。
- 权限是 0~15 十六级且可对单条命令改级（`privilege <模式> level <0-15> <命令>`），命令敲不进去先 `show privilege` 看档位，不要一味怀疑语法。
- `clock set` 参数顺序为「时:分:秒 月 日 年」，与 Cisco `clock set hh:mm:ss MONTH DD YYYY` 接近但不完全等同，写错会被直接拒绝。

## 高危操作 ⚠️

- ⚠️ `delete` 删除 `flash:/cfg/startrun.dat` 等于清掉启动配置：重启后设备回到初始状态，业务全断。删前必须先 `copy` 把 `startrun.dat` 导出到 FTP/TFTP 并确认文件大小正常。
- ⚠️ `write` 是覆盖式保存：错误配置一旦保存，回退只能靠事先导出的 `startrun.dat`。变更前固定动作是「先 `show running-config` + `copy` 导出」，再改。
- ⚠️ 重启/复位类命令（各版本为 `reload` 或 `reboot`，以官方手册确认）会中断全部业务，执行前确认维护窗口、确认配置已保存、确认带外/Console 可用。
- ⚠️ 改 `enable secret`、`username ... privilege`、`banner` 之外的登录相关配置前，先确认还有可用账号与 Console 通道，否则可能把自己锁在设备外。

## 参考来源

1. [中兴官方手册站 · CLI 配置（含 `write` 保存与 startrun.dat 导出）](https://csl.zte.com.cn/manual_zh/topics/1684391333553.html)（官方）
2. [ZXR10 路由交换机基本操作和配置 V2.1 课件（视图/文件系统/权限分级/`write`）](https://www.jinchutou.com/shtml/a85d8cee99dc0393c28ace1e9cf7125b.html)
3. [中兴 M6000-S / 8900E 无法进入 config 模式的处理（140357、`multi-user configure`、`clear line vty`）](https://www.cnblogs.com/ys961206/p/17690916.html)
4. [ZXR10 1800 路由器配置流程（`en` / `config terminal` / `write` 实测链）](http://szdingwei.net/html/6432195423.html)
