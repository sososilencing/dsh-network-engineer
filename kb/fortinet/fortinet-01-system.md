---
vendor: 飞塔（Fortinet）
os: FortiOS
topic: 系统管理与 CLI 层级
devices: FortiGate 40F/60F/70F/100F/600E/1800F 等（FortiOS 6.4 / 7.0 / 7.2 / 7.4 / 7.6）
keywords: [FortiOS, config edit set next end, 配置层级, show full-configuration, 自动保存, config system global, hostname, timezone 时区, config system ntp, VDOM, execute backup config, 配置回退]
sources:
  - https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/105110478/config-system-ntp
  - https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface
  - https://support.fortinet.com.cn/system_mgmt/mgmt_method/cli_mgmt/cli_basics
  - https://support.fortinet.com.cn/system_mgmt/firmware_and_config/config_mgmt/backup_restore
  - https://support.fortinet.com.cn/system_mgmt/firmware_and_config/config_mgmt/factory_reset
verified: high
updated: 2026-10-30
---

# 飞塔（Fortinet）· 系统管理与 CLI 层级

> FortiOS 是"配置节点树"式 CLI：`config <路径>` → `edit <对象>` → `set <参数> <值>` → `next` / `end`，**`next`/`end` 一敲即写入并生效，没有 `write memory`**。查看用 `show`（只显示非默认值）与 `show full-configuration`（含默认值）。它是防火墙，策略默认隐含拒绝。

## 命令速查

| 功能 | 命令 | 视图 | 说明 |
|---|---|---|---|
| 进入配置节点 | `config system global` | 操作模式 `FortiGate #` | `config` 只用于进入配置树，`show/get/diagnose/execute` 前不要加 `config` |
| 编辑一个对象 | `edit port1` / `edit "vlan10"` / `edit 1` | `(global) #` 等节点视图 | 数字对象（路由、策略）用编号，命名对象用引号 |
| 设置参数 | `set hostname FGT-HQ` | `(global) #` | 参数名大小写敏感；`set` 会**替换**整个列表 |
| 结束当前对象 | `next` | 对象视图 `(port1) #` | 保存该对象并继续在同节点 `edit` 下一个 |
| 结束整个配置块 | `end` | 任意配置视图 | 写配置并回到操作模式 |
| 查看配置（精简） | `show system global` | 操作模式 | 只列出与默认值不同的行 |
| 查看配置（全量） | `show full-configuration system global` | 操作模式 | 核对默认值（allowaccess、distance 等）必须用这条 |
| 过滤输出 | `show system interface \| grep allowaccess` | 操作模式 | 管道 `grep` 只做行匹配 |
| 查看命令树 | `tree router static` | 操作模式 | 列出该节点的子命令、字段长度与取值范围 |
| 设主机名与时区 | `set hostname FGT-HQ` / `set timezone "Asia/Shanghai"` | `(global) #` | 7.4 起用字符串时区；7.4 之前用数字索引 `set timezone 55` |
| 时间同步 | `set ntpsync enable` + `config ntpserver` → `edit 1` → `set server "10.1.1.10"` | `(ntp) #` | 字段随版本略有差异，以 `show full-configuration system ntp` 为准 |
| 管理员账号 | `config system admin` → `edit admin` → `set accprofile "super_admin"` | `(admin) #` | 只有 `super_admin` 导出的配置才完整 |
| 切换 VDOM | `config vdom` → `edit vdom1`；全局用 `config global` | 操作模式 | 开启多 VDOM 后配置分"全局层"与"VDOM 层" |
| 备份配置 | `execute backup config tftp <文件名> <服务器IP>` | 操作模式 | 也支持 `ftp` / `sftp` / `usb`，可加加密口令 |
| 恢复配置 | `execute restore config tftp <文件名> <服务器IP>` | 操作模式 | 恢复后**必须重启**才生效 |

## 典型示例

```text
FortiGate # config system global
FortiGate (global) # set hostname FGT-HQ                  // 设备名，等价于华为 sysname
FortiGate (global) # set timezone "Asia/Shanghai"          // FortiOS 7.4+；7.4 之前 set timezone 55
FortiGate (global) # set admin-sport 8443                  // 可选：改 HTTPS 管理端口
FortiGate (global) # end                                   // end 即保存并生效，无需 write memory

FortiGate # config system ntp
FortiGate (ntp) # set ntpsync enable                       // 开启 NTP 客户端
FortiGate (ntp) # set type custom                          // custom=自定义服务器；fortiguard=用飞塔服务器
FortiGate (ntp) # config ntpserver
FortiGate (ntpserver) # edit 1
FortiGate (1) # set server "10.1.1.10"                     // 内网 NTP 服务器地址或域名
FortiGate (1) # next
FortiGate (ntpserver) # end
FortiGate (ntp) # end

FortiGate # config system admin                            // 新建/修改管理员
FortiGate (admin) # edit "netops"
FortiGate (netops) # set accprofile "super_admin"          // 权限模板
FortiGate (netops) # set trusthost1 10.10.10.0 255.255.255.0  // 只允许管理网段登录
FortiGate (netops) # next
FortiGate (admin) # end

FortiGate # show system global                             // 只看非默认值
FortiGate # show full-configuration system ntp             // 含默认值，核对 ntpsync/ntpserver 实际值
FortiGate # get system status                              // 型号/版本/运行模式/当前 VDOM
FortiGate # execute backup config tftp FGT-HQ-20261030.conf 10.1.1.100
FortiGate # execute restore config tftp FGT-HQ-20261030.conf 10.1.1.100  // 恢复后自动重启
```

## 陷阱与版本差异

- **没有 `write memory`/`save`**：`next`/`end` 回车即写 flash。反过来说，改完没 `end` 就断电，配置会丢。
- `show` 只打印与默认值不同的行，核对默认值（接口 `allowaccess`、路由 `distance`、策略 `action`）必须用 `show full-configuration`。
- 时区语法 7.4 是分水岭：`set timezone 55`（数字索引）→ `set timezone "Asia/Shanghai"`（字符串），升级后老脚本会报错。
- `set` 是覆盖式列表赋值：用户组原有 a、b，执行 `set member c` 后只剩 c；追加用 `append member c`，移除用 `unselect member b`。
- 多 VDOM 时 `show` 当前 VDOM 配置，改全局要先进 `config global`；忘了这一层会出现"配置写了但 VDOM 里看不到"。
- 配置文件首行 `#config-version=...` 含型号与版本，手工改配置时不得修改，否则恢复被拒绝；无效命令会被静默忽略，不能只看上传成功。

## 高危操作 ⚠️

- ⚠️ `execute factoryreset`（恢复出厂，会询问 y/n）清空所有配置并自动重启，管理口回到 192.168.1.99、admin 密码置空；⚠️ 执行前必须 `execute backup config` 备份并准备 Console，仅在维护窗口操作。同族还有 `execute factoryreset-shutdown`、`execute factoryreset2`（保留系统设置/接口/VDOM/静态路由），以官方手册为准。
- ⚠️ `execute restore config ...` 覆盖现有全部配置并重启设备；HA 环境主备会同时重启，⚠️ 直接断业务。型号/版本不匹配会被拒绝，但仍须先备份 + 备 Console。
- ⚠️ 修改当前正在使用的接口 IP、`allowaccess`、`admin-sport` 会立刻断开本次管理会话；⚠️ 先建立第二条管理通道（Console 或另一接口）再改。
- ⚠️ 备份文件若设了加密口令，口令遗忘即无法恢复；导出请用 `super_admin` 账户，`prof_admin` 导出的配置缺少 super_admin 账户，导入后无法再导入其他配置。

## 参考来源

1. [config system ntp（docs.fortinet.com，FortiOS 7.2.14 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.2.14/cli-reference/105110478/config-system-ntp)
2. [config system interface（docs.fortinet.com，FortiOS 7.6.8 CLI Reference）](https://docs.fortinet.com/document/fortigate/7.6.8/cli-reference/317104469/config-system-interface)
3. [CLI 基础：命令树、列表增删、环境变量（FortiGate 中文一本通）](https://support.fortinet.com.cn/system_mgmt/mgmt_method/cli_mgmt/cli_basics)
4. [配置备份及恢复：execute backup/restore config（FortiGate 中文一本通）](https://support.fortinet.com.cn/system_mgmt/firmware_and_config/config_mgmt/backup_restore)
5. [恢复出厂：execute factoryreset 与复位键（FortiGate 中文一本通）](https://support.fortinet.com.cn/system_mgmt/firmware_and_config/config_mgmt/factory_reset)
