# dsh-network-engineer · 网络工程师助手

**English** — A DeepSeek Harness plugin that ships a standalone "网络工程师助手" (Network Engineer Assistant) agent preset plus an 80-document command knowledge base covering DPtech (迪普), H3C (新华三), Huawei (华为), Ruijie (锐捷), Cisco IOS/IOS-XE and NX-OS, Juniper Junos, Arista EOS, Fortinet FortiOS and ZTE ZXR10 switches, routers and firewalls: vendor command reference, cross-vendor translation, routing analysis, screenshot reading, and online manual lookup. The assistant only answers networking topics.

---

## 功能

- **预设 Agent「网络工程师助手」**：安装后出现在桌面端/Web 的预设切换器中，选中即用；只回答网络工程主题，其他话题礼貌拒答。
- **内置知识库（80 篇 / 十厂商）**：每厂商 8 主题（系统管理 / VLAN 与二层 / 接口 / 链路聚合 / 静态路由 / OSPF / ACL / 远程管理与诊断），以技能 `network-command-kb` 形式随插件分发，Agent 加载技能后自动获得知识库的安装路径。
- **十厂商命令对照 + 路由优先级差异**（华为 preference 体系 60/10、锐捷 Cisco 风格 AD 1/110、Junos preference 5/10、EOS/NX-OS 的 AD 体系）内置于知识库文档并用于跨厂商对照。
- **联网查手册**：知识库未覆盖的型号/版本差异自动联网检索官方文档并注明来源；禁止编造命令。
- **风险标注**：`reboot` / `format` / `erase` / `restore factory` 等高危命令自动附 ⚠️ 风险提示、备份建议与维护窗口提醒。

## 安装

**方式一（推荐）**：在 DeepSeek Harness 桌面端/Web 安装了 [dshmarket](https://dshmarket.com) 插件市场后，搜索 `dsh-network-engineer` 一键安装（上架后可用）。

**方式二**：DSH CLI/profile 内安装：

```sh
pnpm dsh plugin install dsh-network-engineer   # 或通过 plugin_manager 的 install_bundle
```

**方式三（手动）**：把本目录放进 profile 的 `node_modules/`，并在 profile 的 `cordis.patch.yml` 加一行：

```yaml
- insert:
    - id: network-engineer
      name: dsh-network-engineer
```

安装后重启 dsh，预设切换器选择「网络工程师助手」。

## 使用

1. 新会话选择预设 **网络工程师助手**。
2. 直接提问，例如：「华为交换机怎么把端口划进 VLAN 10」「把这段 H3C 配置转成锐捷」「Cisco Nexus 的 OSPF 为什么邻居起不来」「Junos 的静态路由优先级怎么算」「帮我看这张拓扑图」「分析这条路由为什么不通」。
3. Agent 会先查内置知识库，再联网核实；迪普（公开资料有限）、中兴（官方手册获取受限）相关部分标注 `verified: medium`，回答会提示以官方手册为准。

## 知识库

```
kb/
├── huawei/     华为 VRP（8 篇）
├── h3c/        新华三 Comware V5/V7（8 篇）
├── ruijie/     锐捷 RGOS（8 篇）
├── dptech/     迪普自有 CLI（8 篇）
├── cisco-ios/  思科 IOS / IOS-XE（8 篇）
├── cisco-nxos/ 思科 NX-OS（8 篇）
├── juniper/    瞻博 Junos（8 篇）
├── arista/     Arista EOS（8 篇）
├── fortinet/   飞塔 FortiOS（8 篇）
├── zte/        中兴 ZXR10 ROSNG / ZROS（8 篇）
└── README.md   总览与 RAG 导入指南
```

每篇含 YAML frontmatter（vendor/os/topic/devices/keywords/sources/verified/updated）、命令速查表（功能/命令/视图/说明）、典型示例、陷阱与版本差异、高危操作 ⚠️、参考来源。资料整理自官方手册与高可信社区页面（首批 2026-10-15，新增六厂商 2026-10-30），社区来源中混入的其他厂商命令已隔离进「陷阱与差异」节。

## 上架（维护者）

1. GitHub 建仓并推送本包；在 `package.json` 补 `repository` 字段（市场安装优先"repo-verified npm 包"）。
2. `npm login && npm publish`。
3. 向 [awesome-dsh-plugin](https://github.com/awesome-dsh-plugin/awesome-dsh-plugin) 目录提交 PR（一条条目即可）；[dshmarket](https://dshmarket.com) 市场与网站通常一天内自动收录。

## 目录结构

```
dsh-network-engineer/
├── package.json         # dsh.bundle.patch 声明
├── cordis.patch.yml     # 插件行 + 「网络工程师助手」预设行（含完整 persona 提示词）
├── index.js             # 注册 network-command-kb 技能
├── skills/network-command-kb/SKILL.md
├── kb/                  # 80 篇命令知识库（10 厂商）+ README
├── README.md
└── LICENSE
```

## License

[MIT](LICENSE)
