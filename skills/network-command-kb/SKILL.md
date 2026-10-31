---
name: network-command-kb
description: 十厂商交换机/路由器/防火墙命令知识库（迪普/新华三/华为/锐捷/思科 IOS·NX-OS/Junos/EOS/FortiOS/中兴，80 篇）。回答配置命令、跨厂商语法对照、排障分析前必须先检索本知识库；也用于核对命令语法、默认值与版本差异。
---

# 网络命令知识库使用说明

本技能基础目录下的 `kb/` 子目录是随 `dsh-network-engineer` 插件分发的 80 篇命令知识库，每篇文档头部 frontmatter 含 vendor/os/topic/devices/keywords/sources/verified/updated 元数据。

## 目录结构

- `kb/huawei/`（华为 VRP）、`kb/h3c/`（新华三 Comware）、`kb/ruijie/`（锐捷 RGOS）、`kb/dptech/`（迪普自有 CLI）
- `kb/cisco-ios/`（思科 IOS / IOS-XE）、`kb/cisco-nxos/`（思科 NX-OS）、`kb/juniper/`（瞻博 Junos）、`kb/arista/`（Arista EOS）、`kb/fortinet/`（飞塔 FortiOS）、`kb/zte/`（中兴 ZXR10 ROSNG / ZROS）
- 文件名：`<厂商>-<两位序号>-<主题>.md`；主题序号：01 系统管理、02 VLAN 与二层、03 接口、04 链路聚合、05 静态路由、06 OSPF、07 ACL、08 远程管理与诊断
- `kb/README.md` 为总览、厂商索引、跨厂商选路数值对照与 RAG 导入指南

## 检索方法（用 read/glob/grep 工具）

1. 按「厂商 + 主题」定位文件，例如查华为 OSPF 读 `kb/huawei/huawei-06-ospf.md`，查 NX-OS 的 OSPF 读 `kb/cisco-nxos/cisco-nxos-06-ospf.md`。
2. 跨厂商对照问题同时读多家同名主题文档（如十家的 `*-05-static-route.md`）。
3. 序号主题之外的细节用 grep 按命令关键字跨目录检索（如 grep "port trunk" kb/）。
4. 先确认平台再答：Cisco IOS-XE 与 NX-OS 是两套文档；Juniper 的 EX（交换）与 MX/SRX（路由/安全）语法不同；FortiGate 是防火墙视角。

## 使用规则

- 回答注明来源篇目；frontmatter `sources` 是整理依据，`verified: high` 表示有官方手册来源佐证，`medium` 表示公开实践资料整理。
- `verified: medium` 的篇目（全部 dptech/、全部 zte/、部分 fortinet/）只作方向参考，必须提示用户以官方手册或厂商售后确认为准。
- 知识库未覆盖的主题再联网检索官方文档；任何情况下不编造命令、参数与默认值。
- 文档内标注 ⚠️ 的高危命令（reboot/reload、format、erase、恢复出厂/factoryreset 等）在回答中出现时必须附带风险提示。
