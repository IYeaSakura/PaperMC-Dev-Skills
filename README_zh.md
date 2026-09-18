# PaperMC-Dev-Skills

<div align="center">

[![Minecraft](https://img.shields.io/badge/Minecraft-26.x-62B47A?logo=minecraft)](https://www.minecraft.net/)
[![Paper](https://img.shields.io/badge/Paper-26.2%20stable-2196F3?logo=papermc)](https://papermc.io/)
[![Folia](https://img.shields.io/badge/Folia-26.2%20beta-FF6D00)](https://papermc.io/software/folia)
[![Purpur](https://img.shields.io/badge/Purpur-26.2%20stable-8E44AD)](https://purpurmc.org/)
[![Java](https://img.shields.io/badge/Java-25%20LTS-007396?logo=openjdk)](https://openjdk.org/)
[![Maven](https://img.shields.io/badge/Maven-3.9-C71A36?logo=apache-maven)](https://maven.apache.org/)
[![Gradle](https://img.shields.io/badge/Gradle-8.x-02303A?logo=gradle)](https://gradle.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

[English](README.md) | **中文**

</div>

一套面向 AI 编码 Agent 的生产级技能包，用于开发、调试与维护 **Paper** 及其两大分支 **Folia**、**Purpur** 的 Minecraft 服务端插件，目标版本为 26.x 系列与 Java 25。技能包固化了经核实的版本事实、构建坐标、API 模式、分支特有的线程规则，以及 26.1、26.2、26.3 引入的破坏性变更，使 Agent 无需重新推导生态中快速变化的版本信息即可产出正确的插件代码。

内容基于一手来源撰写：PaperMC 官方文档与公告、带版本号的 Javadoc、Paper/Folia/Purpur 源码、在线下载与 Maven 元数据 API，以及开发者社区的 issue 报告。

[功能特性](#功能特性) | [技术栈](#技术栈) | [项目结构](#项目结构) | [快速开始](#快速开始) | [开发](#开发) | [构建与部署](#构建与部署) | [技能参考](#技能参考) | [版本矩阵](#版本矩阵) | [常见问题](#常见问题) | [贡献指南](#贡献指南) | [许可证](#许可证)

**仓库地址**：[https://github.com/IYeaSakura/PaperMC-Dev-Skills](https://github.com/IYeaSakura/PaperMC-Dev-Skills)

---

## 功能特性

### Paper 26.x 覆盖

- 说明 26.1 之后的新构件版本方案 `<minecraft-version>.build.<number>-<channel>`，以及 `alpha`、`beta`、`stable` 三个通道的官方含义
- 固定当前稳定目标（`paper-api:26.2.build.124-stable`），并标注 26.3 仍处于 alpha
- 结合上游源码引用解释 `api-version` 的解析与校验规则，并给出可能出现的错误信息
- 跟踪 26.1 与 26.2 的破坏性变更：世界存储布局、世界键、按世界计时的时钟、床不再是方块实体、Adventure 5、方块怪重构
- 将已弃用的 Timings 性能分析指引替换为 spark

### Folia 区域化多线程

- 描述 regionizer 模型：独立区域、并行的 tick 循环、按区域计数的 tick 计数器与全局区域
- 说明并强调 `folia-supported: true` 选择性开关，包括 Folia 会拒绝加载未声明的插件这一事实
- 记录全部四个调度器、各自适用场景，以及为何实体操作不能使用区域调度器
- 覆盖线程归属检查（`isOwnedByCurrentRegion`、`isGlobalTickThread`），并给出安全的"就地执行或调度"辅助方法
- 列出 Folia 上已损坏的 API，并明确标注 `Entity#teleport` 永久不可用、应改用 `teleportAsync`
- 提供服务端配置建议，包括推荐的线程分配上限

### Purpur 分支支持

- 说明其与 Paper 的直接替换关系，并确认 Purpur 独有行为默认全部关闭
- 记录完整的 `org.purpurmc.purpur` API：事件、实体、语言与权限工具
- 覆盖基于 Purpur Rebrand 补丁所定义 brand id 的平台检测方式
- 梳理 `purpur.yml` 中可能推翻插件原版假设的全局与世界设置，例如可骑乘生物、被修改的方块行为与属性覆盖
- 演示可选集成模式，使同一个 JAR 仍可在 Paper 上加载

### 跨分支工程

- 提供同一套代码同时运行于 Paper、Purpur、Folia 的单 JAR 方案
- 将 `teleportAsync` 与调度器用法定义为三者之间的可移植性边界
- 给出能力矩阵，明确哪些 API 会牺牲 Folia 兼容性
- 包含隔离钩子模式，避免可选分支 API 缺失时抛出 `NoClassDefFoundError`

### 项目搭建与数据持久化

- 提供可直接复制的完整 Maven 与 Gradle 配置，含 Java 25 toolchain 与 `release` 设置
- 说明 mapping namespace 清单项与 paperweight-userdev dev bundle 工作流
- 覆盖 SQLite、MySQL + HikariCP 与 Caffeine 缓存及其线程安全模式
- 内置面向生产插件的性能与安全检查清单

---

## 技术栈

### 目标平台

| 类别 | 技术 | 版本 |
|------|------|------|
| 基础服务端 | Paper | 26.2（build 124，stable） |
| 分支（线程模型） | Folia | 26.2（build 7，beta） |
| 分支（功能扩展） | Purpur | 26.2（build 2633，stable） |
| 游戏版本 | Minecraft Java Edition | 26.2 |

### 开发工具链

| 类别 | 技术 | 版本 |
|------|------|------|
| 语言 | Java | 25 LTS |
| 构建工具 | Maven | 3.9.x |
| 构建工具 | Gradle | 8.x（Kotlin DSL） |
| 核心 API | Bukkit / Paper API | 26.x |
| 文本库 | Adventure | 5.x |
| NMS 工具 | paperweight-userdev | 2.0.0-beta.23 |

### 可选依赖

| 类别 | 构件 | 用途 |
|------|------|------|
| 缓存 | `com.github.ben-manes.caffeine:caffeine` | 高性能内存缓存 |
| 数据库 | `com.mysql:mysql-connector-j` | MySQL 驱动 |
| 连接池 | `com.zaxxer:HikariCP` | JDBC 连接池 |
| 数据库 | `org.xerial:sqlite-jdbc` | SQLite 驱动（通常由服务端提供） |

### 编写格式

| 类别 | 技术 | 用途 |
|------|------|------|
| 技能格式 | 带 YAML front matter 的 Markdown | 可被 Agent 发现的技能定义 |
| 技能规范 | DSH 渐进式披露技能 | 先加载 `SKILL.md`，按需展开 `references/` |
| 版本控制 | Git | 仓库历史 |

---

## 项目结构

```
PaperMC-Dev-Skills/
├── SKILL.md                     # 技能入口：版本事实、分支矩阵、快速开始
├── references/                  # 渐进式披露的细节，仅在相关时加载
│   ├── project-setup.md         # Maven/Gradle 模板、Java 25 toolchain、分支目标
│   ├── plugin-yml.md            # plugin.yml 与 paper-plugin.yml 规范
│   ├── api-patterns.md          # 事件、命令、调度器、GUI、物品、多目标指南
│   ├── data-storage.md          # SQLite、MySQL/HikariCP、Caffeine、异步模式
│   ├── version-matrix.md        # 版本矩阵、api-version 规则、迁移、NMS
│   ├── folia.md                 # 区域化多线程、调度器、Folia 损坏 API 清单
│   └── purpur.md                # Purpur 分支 API、purpur.yml 选项、安全集成
├── README.md                    # 英文文档
├── README_zh.md                 # 本文件（中文）
└── LICENSE                      # MIT 许可证
```

### 文件职责

| 文件 | 作用 | 大致规模 |
|------|------|----------|
| `SKILL.md` | 始终加载的概要：事实、分支选型、快速开始、检查清单 | 约 400 行 |
| `references/version-matrix.md` | 版本与兼容性的权威数据，含 Paper Family 对比 | 约 640 行 |
| `references/api-patterns.md` | 可运行的 API 示例，以及多目标工程章节 | 约 950 行 |
| `references/folia.md` | 完整的 Folia 指南，含损坏 API 表与迁移清单 | 约 230 行 |
| `references/purpur.md` | 完整的 Purpur 指南，含分支 API 清单与配置覆盖 | 约 270 行 |
| `references/project-setup.md` | 三个目标的构建配置 | 约 430 行 |
| `references/plugin-yml.md` | 元数据文件规范，含 `folia-supported` | 约 330 行 |
| `references/data-storage.md` | 持久化与缓存模式 | 约 380 行 |

---

## 快速开始

### 前置要求

- **兼容 DSH 的 Agent 运行时**，且技能目录对当前会话可用
- **Git** 2.30 或更高版本，用于克隆仓库
- **使用本技能无需任何构建工具链**。只有在根据技能输出真正构建插件时，才需要 Java、Maven 与 Gradle
- 若要构建并测试生成的插件：**Java 25 LTS**，以及 **Maven 3.9+** 或 **Gradle 8+**

### 安装

将技能安装到会话的技能目录中。目录名决定技能在磁盘上的位置；技能的身份来自其 front matter 中的 `name` 字段。

```bash
# Clone the repository
git clone https://github.com/IYeaSakura/PaperMC-Dev-Skills.git

# Install into the session skills directory (Windows default path shown)
# C:\Users\<user>\.dsh\skills\
mv PaperMC-Dev-Skills "%USERPROFILE%\.dsh\skills\Minecraft-Paper-Dev-Skills"
```

```bash
# Linux / macOS equivalent
git clone https://github.com/IYeaSakura/PaperMC-Dev-Skills.git
mv PaperMC-Dev-Skills ~/.dsh/skills/Minecraft-Paper-Dev-Skills
```

### 验证安装

```bash
# Confirm the expected layout is present
ls -1 ~/.dsh/skills/Minecraft-Paper-Dev-Skills
# Expected: LICENSE  README.md  README_zh.md  SKILL.md  references
```

安装完成后，技能会以 `minecraft-paper-dev-skills` 出现在会话技能目录中。可以显式调用，也可以由运行时在请求匹配其描述时自动选用。

### 配置

本技能本身是文档，不需要配置文件或环境变量。以下两项可选调整较为实用：

| 设置项 | 用途 | 说明 |
|--------|------|------|
| 目录名 | 在技能根目录中的放置位置 | 请与运行时约定的技能发现规则核对 |
| 目标版本 | 修改固定的 Paper/Folia/Purpur 构建号 | 需同时修改 `SKILL.md` 与 `references/version-matrix.md` 中的版本事实 |

### 版本快照

技能内置的快照核实日期为 **2026-09-17**。用于新项目前，请先核对在线接口：

```bash
# Paper versions and channel status
curl -s https://fill.papermc.io/v3/projects/paper/versions/26.2/builds | head -c 400

# Folia versions
curl -s https://fill.papermc.io/v3/projects/folia/versions/26.2/builds | head -c 400

# Purpur versions
curl -s https://api.purpurmc.org/v2/purpur/
```

---

## 开发

### 编辑技能内容

所有内容均为 Markdown。为保持整包一致性，请遵循以下约定：

1. `SKILL.md` 保持为概要。把详尽的表格与长示例移到 `references/`，并从 `SKILL.md` 及文件底部的参考表链接过去。
2. 版本事实发生变化时，同步更新 `SKILL.md` 中的快照日期。
3. 各 reference 之间互相链接，使读者能在概要与细节之间往返跳转。
4. 任何新增的事实性论断都要标注一手来源。

### 内容约定

| 项目 | 约定 | 示例 |
|------|------|------|
| 标题 | 句首大写，指导性内容用祈使式 | `## Marking a Plugin as Folia-Supported` |
| 代码块 | 必须标注语言 | `java`、`kotlin`、`yaml`、`bash` |
| 版本字符串 | 使用反引号，采用精确构件写法 | `26.2.build.124-stable` |
| 类引用 | 首次出现时使用全限定名 | `io.papermc.paper.threadedregions.scheduler.RegionScheduler` |
| 平台名称 | 按品牌大小写书写 | Paper、Folia、Purpur |
| 弃用论断 | 明确给出替代方案 | `Use teleportAsync` |
| 表格 | 用于矩阵与对比 | 版本表、能力表、配置表 |

### 校验流程

由于技能固化了快速变化的版本事实，修改后应对照在线来源校验：

```bash
# 1. Confirm the Paper API artifact still resolves
curl -s https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml \
  | grep -o '<version>26\.[0-9.]*build\.[0-9]*-stable</version>' | tail -n 3

# 2. Confirm the Purpur API artifact still resolves
curl -s https://repo.purpurmc.org/snapshots/org/purpurmc/purpur/purpur-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3

# 3. Confirm the Folia API artifact still resolves
curl -s https://repo.papermc.io/repository/maven-public/dev/folia/folia-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3
```

### 评审清单

提交内容变更前：

- [ ] 任何改动的版本号都已对照在线接口或带版本号的 Javadoc 核实
- [ ] 新增的 API 论断指明具体类名与方法名，而非转述
- [ ] 破坏性变更说明其属于源码破坏、二进制破坏还是弃用
- [ ] 分支特有论断标明其适用的服务端
- [ ] `SKILL.md` 仍为概要，未重复整段 reference 内容
- [ ] 版本快照日期已反映最近一次核实

---

## 构建与部署

本项目没有编译步骤；"构建"指打包，"部署"指分发给技能的使用方。

### 生成分发包

```bash
# Create a release archive containing only tracked files
git archive --format=zip --output=PaperMC-Dev-Skills.zip HEAD

# Or produce a tarball
git archive --format=tar.gz --output=PaperMC-Dev-Skills.tar.gz HEAD
```

### 部署目标

| 目标 | 方式 | 说明 |
|------|------|------|
| 本地 Agent 会话 | 复制到会话技能目录 | 主要使用方式 |
| 团队分发 | 提交到内部镜像 | 必须包含 `references/` 目录 |
| 公开发布 | 发布到 GitHub 并打标签 | 使用方克隆到自己的技能根目录 |
| 离线环境 | 分发发布归档包 | 阅读技能无需网络访问 |

### 发布流程

```bash
# Tag a release after content updates
git tag -a v26.2.0 -m "Skill content updated for Paper 26.2 / Folia 26.2 / Purpur 26.2"
git push origin v26.2.0
```

### 建议的版本号规则

| 组成部分 | 含义 | 示例 |
|----------|------|------|
| 主版本 | 保留给技能结构的重构 | `1.0.0` |
| 次版本 | 新增平台覆盖或新增 reference 文件 | `0.2.0` |
| 修订号 | 事实性修正与版本号更新 | `0.2.1` |

---

## 技能参考

### 技能元数据

| 字段 | 值 |
|------|-----|
| 技能名称 | `minecraft-paper-dev-skills` |
| 入口文件 | `SKILL.md` |
| 调用方式 | 按名称显式调用，或按描述匹配自动选用 |
| 加载模型 | 渐进式：先 `SKILL.md`，再按需加载 `references/` |

### Reference 文件索引

| 主题 | 文件 | 内容 |
|------|------|------|
| Paper 26.x 概览与快速开始 | `SKILL.md` | 版本事实、分支矩阵、项目搭建、检查清单 |
| 构建配置 | `references/project-setup.md` | Maven 与 Gradle 模板、Java 25 toolchain、分支依赖 |
| 插件元数据 | `references/plugin-yml.md` | `plugin.yml` 与 `paper-plugin.yml`、`api-version`、`folia-supported` |
| API 模式 | `references/api-patterns.md` | 事件、命令、调度器、GUI、物品、实体、多目标指南 |
| 数据持久化 | `references/data-storage.md` | SQLite、MySQL、HikariCP、Caffeine、异步桥接 |
| 版本与迁移 | `references/version-matrix.md` | 版本矩阵、Paper Family 对比、`api-version` 规则、NMS |
| Folia | `references/folia.md` | 区域化多线程、调度器、线程归属、损坏 API |
| Purpur | `references/purpur.md` | 分支 API、`purpur.yml`、权限、安全集成 |

### 版本探测参考

| 服务端 | 最新版本接口 | 构件坐标 |
|--------|--------------|----------|
| Paper | `https://fill.papermc.io/v3/projects/paper` | `io.papermc.paper:paper-api` |
| Folia | `https://fill.papermc.io/v3/projects/folia` | `dev.folia:folia-api` |
| Purpur | `https://api.purpurmc.org/v2/purpur/` | `org.purpurmc.purpur:purpur-api` |

---

## 版本矩阵

截至 **2026-09-17** 快照：

| PaperMC | Minecraft | 最低 Java | api-version | 状态 |
|---------|-----------|-----------|-------------|------|
| 1.20.5 - 1.20.6 | 1.20.5 - 1.20.6 | 21 | `1.20` | 历史版本 |
| 1.21 - 1.21.11 | 1.21 - 1.21.11 | 21 | `1.21` | 历史版本 |
| 26.1.1 | 26.1.1 | 25 | `26.1` | 已停止支持 |
| 26.1.2 | 26.1.2 | 25 | `26.1` 或 `26.1.2` | 受支持 |
| **26.2** | **26.2** | **25** | **`26.2`** | **最新稳定版** |
| 26.3 | 26.3 | 25 | `26.3` | 仅 Paper alpha |

### 分支对比

| | Paper | Folia | Purpur |
|---|---|---|---|
| 关系 | 基础服务端 | 加入区域化多线程的 Paper 分支 | Paper 的直接替换版 |
| 线程模型 | 单一主线程 | 无主线程；每个区域一个 tick 循环 | 单一主线程 |
| 26.2 最新构建 | `124-stable` | `7-beta` | `2633-stable` |
| Maven group | `io.papermc.paper` | `dev.folia` | `org.purpurmc.purpur` |
| 预发布通道名 | `alpha` | `beta` | `experimental` |
| 是否需要插件声明 | 否 | 需要 `folia-supported: true` | 否 |
| 普通 Paper 插件能否加载 | 可以 | 几乎都不能 | 可以，行为不变 |

---

## 常见问题

### 技能未被发现

**问题**：会话技能目录中看不到该技能。

**解决方法**：
- 确认技能目录直接位于技能根目录下，而非多嵌套了一层
- 确认技能目录根存在 `SKILL.md`，且其 front matter 同时包含 `name` 与 `description`
- 确认目录命名符合运行时的技能发现约定

### Reference 文件未被读取

**问题**：Agent 只依据 `SKILL.md` 作答，从不打开 `references/`。

**解决方法**：
- 检查 `SKILL.md` 中的相对链接是否可解析，例如 `references/folia.md`
- 显式点名主题，例如"阅读 Folia 参考文档并按其修改"
- 保持 `SKILL.md` 精简；过于臃肿的入口会抑制 reference 展开

### 版本事实过期

**问题**：技能推荐的构建号已不是最新稳定版。

**解决方法**：
- 重新查询[版本快照](#版本快照)中的接口
- 更新 `SKILL.md` 中的版本事实与 `references/version-matrix.md` 中的矩阵
- 更新快照日期并提交

### 目标分支选择错误

**问题**：把 Folia 的建议套用到 Paper 或 Purpur 上，或反之。

**解决方法**：
- 应用调度器或线程相关建议前，先查看上方的分支对比表
- 把 Folia 的限制视为叠加项：在 Folia 上损坏的 API 在 Paper 与 Purpur 上依然有效
- 记住 Purpur 继承 Paper 的单线程模型，并非 Folia 分支

### 服务端拒绝 api-version

**问题**：服务端日志出现 `Unsupported API version`，或拒绝加载插件。

**解决方法**：
- 使用 `major.minor` 形式如 `26.2`，而非构建号如 `26.2.build.124-stable`
- 不要使用 `1.26.x` 形式；2026 年的方案是 `year.drop`
- 若插件因过旧被拒绝，检查 `bukkit.yml` 中的 `settings.minimum-api`

---

## 贡献指南

欢迎贡献。版本事实的准确性是首要质量标准。

1. Fork 本仓库
2. 创建特性分支：`git checkout -b feature/your-topic`
3. 按上方内容约定进行修改
4. 对照在线接口或带版本号的 Javadoc 核实每一个版本号
5. 提交：`git commit -m 'docs: update Paper 26.x version facts'`
6. 推送：`git push origin feature/your-topic`
7. 发起 Pull Request

### 代码质量要求

提交 PR 前：

- [ ] 每个版本号均已对照权威来源核实
- [ ] 每个 API 论断都指明了真实的类名与方法名
- [ ] 分支特有指导均标明适用的服务端
- [ ] 破坏性变更已归类为源码破坏、二进制破坏或弃用
- [ ] `SKILL.md` 中的链接均指向存在的文件
- [ ] 若版本事实有变动，已更新快照日期
- [ ] 正文中未使用 emoji

---

## 许可证

This project is licensed under the MIT License. See [LICENSE](LICENSE) file for details.

```
MIT License

Copyright (c) 2026 Yuyang.Wang

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 致谢

本技能由其所覆盖项目的文档与源码汇编而成：

- [PaperMC](https://papermc.io/) - Paper 服务端、开发文档与带版本号的 Javadoc
- [Folia](https://github.com/PaperMC/Folia) - 区域化多线程模型及其公布的损坏 API 清单
- [PurpurMC](https://purpurmc.org/) - Purpur 分支、配置文档与 Javadoc
- [Adventure](https://docs.advntr.dev/) - Paper 26.2 随附的组件化文本 API
- [paperweight](https://github.com/PaperMC/paperweight) - 用于服务端内部开发的 Gradle 工具
- [Mojang Studios](https://www.minecraft.net/) - Minecraft Java Edition

---

## 联系方式

- **作者**：Yuyang.Wang
- **网站**：[https://sakurain.net](https://sakurain.net)
- **邮箱**：[Yae_SakuRain@outlook.com](mailto:Yae_SakuRain@outlook.com)
- **GitHub**：[https://github.com/IYeaSakura](https://github.com/IYeaSakura)

---

<p align="center">
  Made by Yuyang.Wang
</p>
