# 把一个领域的知识装进 DeepSeek Harness：我写了一个 4000 行的 Minecraft 插件开发 Skill

去年七月我给 DeepSeek Harness（后面简称 dsh）写了一个 Skill，用来开发 Minecraft 服务端插件。当时的想法很朴素：我平时要查的东西太碎了。

Paper 服务端的 API 变动频繁，版本号从 2026 年开始还换了一套体系。以前是 `1.21.4` 这种写法，现在变成 `26.2`，中间还夹着构建号、发布通道、`api-version` 字段的一堆规则。问模型几次之后我发现它在这些地方错得很有规律，而且错得很自信。

于是就有了这个项目。中间推翻重写过一次，现在是 8 个文件、4620 行（含双语 README），仓库在 Gitee：`gitee.com/IYeaSakura/PaperMC-Dev-Skills`。

这篇不是 Harness 的介绍文。我想讲的是具体怎么把一个专业领域的知识组织成 Skill，哪些地方容易踩坑，以及我核事实的流程。

## 先说清楚 dsh 的 Skill 是怎么加载的

这部分我是直接读 `node_modules/@deepseek-ai/dsh-skill-filesystem` 的包文档确认的，不是凭印象写的。

Skill 有两种放法，都放在被扫描的根目录下：

```
<root>/<name>/SKILL.md      ← 目录 bundle
<root>/<name>.md            ← 平铺文件
```

有个约束我一开始没料到：**只扫一层**。你写 `<root>/a/b/SKILL.md` 是找不到的，嵌套的 `**/SKILL.md` 刻意不支持。我本来想按主题再分一层目录，试了才发现不行，只好改成文件名带前缀来区分。

`SKILL.md` 开头是 YAML frontmatter，字段不多：

| 字段 | 是否必填 | 说明 |
|---|---|---|
| `name` | 必填 | 必须是 kebab-case |
| `description` | 必填 | 渲染进会话目录，有长度上限 |
| `whenToUse` | 可选 | 补充使用时机 |
| `metadata` | 可选 | 自定义元数据 |
| `disable-model-invocation` | 可选 | 设为 true 则模型看不到这个 Skill |
| `user-invocable` | 可选 | 设为 false 则用户命令里不出现 |

布尔值的写法比较宽容，`true`/`yes`/`on`/`1` 都认。但有个坑：**拼错了整个 Skill 会被静默丢弃**，只留一条警告。模型侧拿不到任何逐条诊断，你只能看到 Skill 没出现，看不出是文件名错了、frontmatter 格式错了还是布尔值写错了。我第一次调试时在这上面花了点时间。

扫描的根目录有优先级，数值越小越优先：

| Rank | 来源 | 路径 |
|---|---|---|
| 100 | project-dsh | `<projectRoot>/.dsh/skills` |
| 200 | project-agents | `<projectRoot>/.agents/skills` |
| 300 | custom | `Config.customSkillDirs` |
| 400 | user-dsh | `<dshHome>/skills` |
| 500 | user-agents | `<agentsHome>/skills` |

项目根的定义是"最近的包含 `.git` 的祖先目录"。同名 Skill 跨层时近的胜出，也就是说项目里可以放一个同名 Skill 覆盖用户级的。

## 三个决定了整体结构的机制

读文档时有三条我一开始没在意，后来发现它们直接决定了 Skill 该怎么组织。

**目录和正文是分开的。** 发现阶段只解析 frontmatter 生成目录条目，加载时才重新读文件正文。这意味着改正文不需要任何缓存失效操作，存盘即生效。

**根目录有 watcher。** 新增、改名、删除 Skill 都不用重启 dsh，下一个模型步骤就能看到。

**`references/` 目录下的改动不触发目录刷新。** 只有直属的 `SKILL.md` 增删改才会。这一条比较关键：它意味着你把大块内容放进子目录之后，编辑这些内容不会引起会话目录的重建。

第三条直接决定了我后来的文件结构。

## 为什么我没有把所有东西写进一个文件

最开始的版本就是一个大 `SKILL.md`，280 多行。写到后来发现两件事。

一是 `description` 字段是常驻成本。它会被渲染进会话的初始目录，每个会话都吃一遍。我现在的 `description` 是 919 个字符，已经不短了，但这只是目录条目，不是全文。

二是如果把 4000 行全部堆进 `SKILL.md`，那就是每个会话都加载 4000 行。Skill 的价值本来就在于按需加载，这么写等于把按需加载变成了每次全量加载。

所以改成了索引加分册的结构：

```
Minecraft-Paper-Dev-Skills/
├── SKILL.md                     401 行    ← 常驻索引
├── references/
│   ├── api-patterns.md          952 行    ← 事件、命令、调度器、GUI、物品
│   ├── version-matrix.md        643 行    ← 版本矩阵、api-version 规则、迁移
│   ├── project-setup.md         408 行    ← Maven/Gradle 模板、三平台依赖
│   ├── data-storage.md          402 行    ← SQLite、MySQL、缓存
│   ├── plugin-yml.md            328 行    ← 元数据文件规范
│   ├── purpur.md                267 行    ← Purpur 分支
│   └── folia.md                 231 行    ← Folia 区域化多线程
├── README.md / README_zh.md     494 + 494 行
└── LICENSE
```

索引和分册的比例大概是 1:8。`SKILL.md` 只放四类东西：

- 版本事实速查（当前稳定版是哪个构建号）
- 三个平台（Paper / Folia / Purpur）的选型矩阵
- 快速开始的骨架代码
- 性能和安全检查清单

判断某段内容该不该留在 `SKILL.md`，我用一个很简单的标准：**如果 80% 的会话都用不到它，就挪进 references。**

## 核事实的部分，才是这个项目真正花时间的

Minecraft 插件生态有个特殊情况，版本号体系 2026 年刚换过，模型语料里全是旧的 `1.21` 经验。你没法通过在提示词里多写一句"注意版本可能已更新"来解决，因为模型不知道自己不知道。

所以我给每条事实定的规矩是必须能追溯到一手来源，优先级是：**源码 > 官方文档 > 元数据接口 > 社区讨论**。

### 第一个例子：`api-version` 到底该填什么

我最初查到的资料说填 `26.1.2`。差点就这么写进去了，后来觉得不放心，去翻了 Paper 自己的解析代码：

```java
// paper-server: org.bukkit.craftbukkit.util.ApiVersion
String[] versionParts = versionString.split("\\.");
if (versionParts.length != 2 && versionParts.length != 3) {
    throw new IllegalArgumentException(
        "API version string should be of format \"major.minor.patch\" or \"major.minor\"…");
}
int major = parseNumber(versionParts[0]);
int minor = parseNumber(versionParts[1]);
```

它按 `major.minor.patch` 解析。所以：

- `26.2` → major=26, minor=2, patch=0
- `26.1.2` → major=26, minor=1, patch=2

**后者在语义上比前者小。** 一个插件声明 `api-version: '26.1.2'`，在 26.2 服务器上虽然能跑（因为比当前版本旧），但如果它用了 26.2 才有的 API，声明就是错的，而且 26.1.2 这个写法会让本该拒绝的服务器放行。

为了确认，我又去看了 Paper 仓库的 `gradle.properties`，那里面写得很直白：

```properties
mcVersion=26.2
apiVersion=26.2   # the current API version for use in (paper-)plugin.yml files
channel=STABLE
```

结论是 `'26.2'`，我原来查到的资料是错的。这件事让我在后面所有版本号上都不敢偷懒，**凡是版本语义，一律去读解析它的那段代码。**

### 第二个例子：`folia-supported` 不是"声明"，是"开关"

Folia 是 Paper 的区域化多线程分支，每个区域有独立的 tick 循环并行执行，没有主线程这个概念。我给这个分支单独写了 231 行的 reference。

写之前我以为 `folia-supported: true` 是一个"告诉用户我兼容 Folia"的标记字段。读了 Folia 的 README 才发现性质不同：

> only plugins that have been explicitly marked by the author(s) to work with Folia will be loaded

**不加这个字段，Folia 直接不加载你的插件。** 不是警告，不是降级运行，是插件列表里根本不会出现。

这两个理解的差别很大。前者你会想"加不加都行"，后者你必须先真的改完代码再加。Folia 的 README 里还有一句更直接的：对未修改的 Paper 插件，兼容性期望是 0。

### 第三个例子：Purpur 的检测写法

Purpur 是 Paper 的直接替换版，加了一堆默认关闭的玩法配置。如果你要在插件里用它的独有 API，就得先判断当前跑的是不是 Purpur。

我第一版写的是字符串比较：

```java
ServerBuildInfo.buildInfo().brandName().equalsIgnoreCase("Purpur")
```

能用，但总觉得别扭。后来去翻 Purpur 仓库的补丁目录，发现它有一个 Rebrand 补丁，内容很短：

```java
// Purpur start
/**
 * The brand id for Purpur.
 */
Key BRAND_PURPUR_ID = Key.key("purpurmc", "purpur");
// Purpur end
```

它给 `ServerBuildInfo` 加了官方常量。正确的判断方式是 `isBrandCompatible()`。

但这里还有个细节：如果你为了多平台兼容而编译对 `paper-api`，这个常量是不存在的（它只在 `purpur-api` 里）。所以我给出的写法是本地构造 Key，两边都能编译：

```java
private static boolean isPurpur() {
    // 与 Purpur 补丁定义的一致；本地构造，paper-api 下也能编译
    return ServerBuildInfo.buildInfo()
        .isBrandCompatible(Key.key("purpurmc", "purpur"));
}
```

能被 API 表达的判断，不要退化成字符串比较。

## 三个平台的差异怎么组织

Paper、Folia、Purpur 行为差别不小，但显然不能让模型每次都读三份完整文档。

我的做法是主文件放对比矩阵，分册放各自细节。矩阵里最关键的是三件事：Maven 坐标、线程模型、以及前面说的加载开关。

```kotlin
// Paper（默认目标）
compileOnly("io.papermc.paper:paper-api:26.2.build.124-stable")

// Folia（注意 group 是 dev.folia，不是 io.papermc.paper）
compileOnly("dev.folia:folia-api:26.2.build.7-beta")

// Purpur（仓库在 repo.purpurmc.org，且 purpur-api 已包含 Paper 的 API）
compileOnly("org.purpurmc.purpur:purpur-api:26.2.build.2633-stable")
```

| 项目 | Paper | Folia | Purpur |
|---|---|---|---|
| 线程模型 | 单主线程 | 无主线程，按区域并行 | 单主线程 |
| 最新 26.2 构建 | 124-stable | 7-beta | 2633-stable |
| 预发布通道名 | alpha | beta | experimental |
| 插件需要额外声明 | 否 | `folia-supported: true` | 否 |
| 普通 Paper 插件能否加载 | 可以 | 基本不行 | 可以，行为不变 |

比矩阵更有用的是**能力矩阵**。与其写一段"请注意 Folia 不支持某些 API"的说明文，不如给一张表让模型直接查：用了 `Bukkit.getScheduler()` → Folia 不可用；用了计分板 API → Folia 不可用；用了 `entity.teleport()` → 换成 `teleportAsync()` 就能三平台通吃。

这里有个细节：Paper 上也提供了 Folia 那四个调度器（`getGlobalRegionScheduler`、`getRegionScheduler`、`getAsyncScheduler`、`Entity#getScheduler`），行为是"投递到主线程"。所以**只要全程用这四个调度器和 `teleportAsync`，一份 JAR 就能同时跑在三个平台上**，编译目标还是 `paper-api`。

分支独有的 API 我用"隔离类 + 品牌检测"包起来，避免 `NoClassDefFoundError` 把整个插件在 Paper 上拖死：

```java
// PurpurHooks.java，只有确认是 Purpur 才会被加载
final class PurpurHooks {
    static void register(JavaPlugin plugin) {
        plugin.getServer().getPluginManager()
            .registerEvents(new PurpurBeeListener(), plugin);
    }
}

// onEnable
if (isPurpur()) {
    try {
        PurpurHooks.register(this);
    } catch (NoClassDefFoundError e) {
        getLogger().warning("Purpur API missing: " + e.getMessage());
    }
}
```

## 时效性得写进文件本身

这类技术文档有个绕不开的问题：写的时候就注定会过期。

我的处理是三层。全文标注快照日期（现在是 2026-09-17）；给出可执行的复查命令，让读者能自己更新；在 README 的常见问题里专门写一节"版本事实过期了怎么办"。

复查命令是这几个，直接可以跑：

```bash
# Paper：列出最新的 stable 构建
curl -s https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml \
  | grep -o '<version>26\.[0-9.]*build\.[0-9]*-stable</version>' | tail -n 3

# Folia
curl -s https://repo.papermc.io/repository/maven-public/dev/folia/folia-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3

# Purpur
curl -s https://repo.purpurmc.org/snapshots/org/purpurmc/purpur/purpur-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3
```

这里还发现一个容易踩的坑。Paper 的 Maven 元数据里，`<latest>` 和 `<release>` 两个标签指向的都是 `26.3-pre-2.build.0-alpha`，是个 alpha 版。**它们并不指向最新稳定版。** 想拿稳定版得自己解析版本列表，筛 `-stable` 后缀。如果照着 `<release>` 写依赖，拿到的就是预发布版。

## 我改了两次

从提交记录能看出整个过程：

```
17e701e  2026-07-06  first commit
a778282  2026-09-17  Update skill for Paper 26.x（26.2 stable / 26.3 alpha）
173af59  2026-09-18  Add Folia and Purpur fork coverage
2df7a0c  2026-09-18  Purpur: document the official brand id
9d014e5  2026-09-18  Add bilingual README documentation and LICENSE
```

第一次提交只覆盖 26.1.2 一个版本，内容也比较浅。两个多月后整体重写了一遍，把 26.1 和 26.2 的破坏性变更补全，又加了 Folia 和 Purpur 两个分支。

我不觉得这说明第一版做得差。领域知识类的 Skill 本来就该是活的，模型换代、API 变动、你自己的理解加深，都会让它需要重写。**重要的是把更新流程写进文档，而不是指望一次写对。**

## 用这套方法做什么都行

把 Minecraft 的具体内容抽掉，流程是这七步：

1. **定边界。** 先想清楚不写什么。我一开始想把服务端运维配置也写进去，后来砍掉了，因为那会让 references 无限膨胀。
2. **建索引。** `SKILL.md` 只放高频内容：速查表、选型矩阵、检查清单、骨架代码。
3. **分册。** 一个主题一个文件放 `references/`，文件名要能一眼看出内容。
4. **核事实。** 定来源优先级，每条可验证的结论都留溯源路径。这一步最花时间，也最值钱。
5. **标时效。** 全局快照日期加局部复查命令，别让读者去猜这份文档是哪个年代的。
6. **写检查清单。** 把你自己做判断时的隐性标准显式化，比如"破坏性变更必须标注是源码破坏还是二进制破坏"。
7. **设计更新机制。** 明确写清楚版本推进时该改哪几个文件。

## 怎么装

```bash
git clone https://gitee.com/IYeaSakura/PaperMC-Dev-Skills.git
mv PaperMC-Dev-Skills ~/.dsh/skills/Minecraft-Paper-Dev-Skills
ls ~/.dsh/skills/Minecraft-Paper-Dev-Skills
# LICENSE  README.md  README_zh.md  SKILL.md  references
```

装完在会话里叫 `minecraft-paper-dev-skills` 就能用。它会以 401 行的索引进入上下文，你需要哪块细节时再去读对应的 references 文件。

如果你想改成自己领域的 Skill，README 里有改造清单，核心就是上面那七步，另外两点容易忽略：`description` 要当广告位写（它决定模型会不会调用你），以及文件名前缀要能替代目录分层（因为只扫一层）。

---

仓库地址：`https://gitee.com/IYeaSakura/PaperMC-Dev-Skills`（GitHub 镜像 `IYeaSakura/PaperMC-Dev-Skills`）

文档里所有版本号都标注了核实日期和溯源方式，如果你要拿去用，建议先跑一遍上面那三条 curl 确认当前构建号。

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
