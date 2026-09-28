> 仓库地址：`https://gitee.com/IYeaSakura/PaperMC-Dev-Skills`，MIT 协议，持续更新中。

DeepSeek Harness（下称 dsh）是 DeepSeek 开源的本地 Agent 运行框架，其技能（Skill）机制允许在会话中按需加载外部指令文档。该机制适合承载特定领域内需要精确、可溯源的知识。

本文介绍我近期制作的一个面向 Minecraft 服务端插件开发的技能包 **Minecraft-Paper-Dev-Skills**，内容覆盖 Paper 26.x 及其两个分支 Folia、Purpur。技能包的技能文档共 3600 余行，采用 `SKILL.md` 加 `references/` 的分层结构。

## 一、问题定义

Paper 插件开发涉及的信息源较为分散：字段取值范围需要查阅官方 Javadoc，接口语义需要查阅服务端源码，依赖版本需要查询 Maven 元数据。当版本体系发生变更时，这些信息还需要交叉验证。

Minecraft Java 版的版本编号体系在 2026 年发生变更，由 `1.21.x` 形式改为 `26.x` 形式。该变更同时影响插件元数据字段取值、依赖坐标格式、构建通道命名与世界存档目录结构。通用大语言模型在该领域的输出因此存在系统性偏差：训练语料中的插件开发经验以旧版本体系为主，而模型无法判断自身知识的时效边界。

下表对比两类信息源的输出差异：

| 查询内容 | 通用模型输出 | 本技能包输出 |
|---|---|---|
| Paper 26.2 的 `api-version` 取值 | `26.1.2` 或 `1.21` | `'26.2'`，依据 `ApiVersion` 解析逻辑说明 `26.1.2` 的语义低于 `26.2` |
| 插件能否运行于 Folia | 需进行兼容性适配 | 需检查 `folia-supported` 字段，未声明时 Folia 不加载该插件 |
| 26.2 升级需修改的接口 | 提示注意 API 变更 | 列出 9 项弃用项与 3 项破坏性变更，并给出迁移代码 |
| 当前最新稳定构建号 | 无可靠来源，可能编造 | `129-stable`，并附版本核验命令 |

技能包的作用是将上述精确信息固化为可检索的文档，使模型的输出从推测转为引用。

## 二、dsh 的技能加载机制

技能包的目录结构与内容组织方式由 dsh 的加载机制决定。以下三条约束来自 `@deepseek-ai/dsh-skill-filesystem` 与 `@deepseek-ai/dsh-tool-skill` 的包文档。

**发现规则**。技能有两种放置形式，均位于被扫描的根目录下：

```
<root>/<name>/SKILL.md      目录 bundle
<root>/<name>.md            平铺文件
```

发现深度为一层，不支持嵌套的 `**/SKILL.md`。`SKILL.md` 需以 YAML frontmatter 开头，其中 `name`（kebab-case）与 `description` 为必填字段，另有可选的 `whenToUse`、`metadata` 与两个调用控制字段。

**目录与正文分离**。会话目录只包含技能的规范化名称与描述；完整指令正文仅在模型显式加载时才读取。这意味着目录条目是常驻上下文开销，正文是一次性开销。`dsh-tool-skill` 的 `catalogDescriptionMaxLength` 配置项默认值为 500，即目录中渲染的描述超过 500 字符会被截断。

**用户级根目录**。扫描根目录按 rank 排序，用户级的 `<dshHome>/skills`（即 `~/.dsh/skills`）位于 rank 400。项目级根目录 `<projectRoot>/.dsh/skills` 的 rank 为 100，同名技能以较近的层优先。

上述机制直接约束了技能包的两个设计决策：描述必须压缩到 500 字符以内且把关键能力前置；正文必须拆分，不能全部写入索引。

## 三、结构设计

技能包的技能文档共 3600 余行，文件构成如下：

```
Minecraft-Paper-Dev-Skills/
├── SKILL.md                     401 行   常驻索引
├── references/
│   ├── api-patterns.md          952 行   事件、命令、调度器、GUI、物品、实体
│   ├── version-matrix.md        643 行   版本矩阵、api-version 规则、迁移指南
│   ├── project-setup.md         408 行   Maven/Gradle 配置、三平台依赖坐标
│   ├── data-storage.md          402 行   SQLite、MySQL、HikariCP、Caffeine
│   ├── plugin-yml.md            328 行   plugin.yml 与 paper-plugin.yml 字段规范
│   ├── purpur.md                267 行   Purpur 专有 API 与配置影响
│   └── folia.md                 231 行   Folia 线程模型与不可用 API 清单
├── README.md / README_zh.md     494 + 494 行
└── LICENSE
```

`SKILL.md` 作为常驻索引，只保留四类高频内容：版本事实速查表、三平台选型矩阵、快速开始骨架代码、性能与安全检查清单。低频细节按主题拆分至各 reference 文件。索引与正文的比例约为 1:8。

描述字段的取值精确定位在 498 字符，低于 500 的上限，且将"支持 Paper、Folia、Purpur 三平台"与"Java 25"置于句首，避免截断时丢失关键能力项。

## 四、内容准确性验证

技能包中每条结论均可追溯至一手来源，来源优先级为：服务端源码、官方文档、元数据接口。以下列举三个典型案例。

### 4.1 `api-version` 字段取值

Paper 对该字段的解析逻辑位于 `org.bukkit.craftbukkit.util.ApiVersion`：

```java
String[] versionParts = versionString.split("\\.");
if (versionParts.length != 2 && versionParts.length != 3) {
    throw new IllegalArgumentException(
        "API version string should be of format \"major.minor.patch\" or \"major.minor\"…");
}
int major = parseNumber(versionParts[0]);
int minor = parseNumber(versionParts[1]);
int patch = versionParts.length == 3 ? parseNumber(versionParts[2]) : 0;
```

解析按 `major.minor.patch` 三段进行，因此：

| 取值 | 解析结果 |
|---|---|
| `26.2` | major=26, minor=2, patch=0 |
| `26.1.2` | major=26, minor=1, patch=2 |
| `1.21` | major=1, minor=21, patch=0 |

`26.1.2` 与 `26.2` 并非同一版本的补丁关系，其语义低于 `26.2`。在 26.2 服务端声明 `api-version: '26.1.2'` 虽可通过加载校验，但若插件调用了 26.2 新增接口，该声明将导致服务端跳过应当执行的版本检查。

Paper 仓库的 `gradle.properties` 给出了该字段的期望取值：

```properties
mcVersion=26.2
apiVersion=26.2   # the current API version for use in (paper-)plugin.yml files
channel=STABLE
```

结论：26.2 服务端应声明 `'26.2'`。若需同时兼容 26.1 与 26.2，可声明 `'26.1'`，前提是未使用 26.2 新增接口。

补充约束：服务端 `settings.minimum-api`（默认 `none`）为取值下限，低于该值的插件将被拒绝加载。

### 4.2 26.1 至 26.2 的破坏性变更

文档将该区间的变更分为两类：9 项弃用项（保留可用替代方案，编译通过但应修改）与 3 项破坏性变更（编译失败或运行时报错）。

破坏性变更之一为床的方块实体移除。26.2 中床变为普通方块，不再支持 `PersistentDataContainer`。`org.bukkit.block.Bed` 已标记弃用，`setColor()` 抛出 `UnsupportedOperationException`。原有数据需在数据修复阶段迁移：

```java
@EventHandler
public void onBlockEntityRemoved(AsyncServerDataFixerRemoveBlockEntityEvent event) {
    if (!event.getBlockEntityType().equals(Key.key("minecraft", "bed"))) return;

    PersistentDataContainerView pdc = event.getPersistentDataContainerView();
    String myData = pdc.get(new NamespacedKey(this, "my_key"), PersistentDataType.STRING);
    if (myData == null) return;

    // 该事件在区块加载流程中触发，可能位于 worker 线程
    // 耗时操作应移交独立线程池
    getServer().getAsyncScheduler().runNow(this, task ->
        getLogger().info("迁移数据: " + myData));
}
```

需注意该事件虽以 `Async` 为前缀，实际执行位置为区块加载流程，可能位于 worker 线程或主线程。Javadoc 明确说明阻塞操作将影响服务端运行，主线程可能因此等待 worker 完成。

破坏性变更之二为 Adventure 5 引起的 `BookMeta` 接口变更，该变更会导致运行期 `NoSuchMethodError`：

```java
// 26.1 写法，26.2 无法编译
BookMeta built = meta.toBuilder().title(Component.text("指南")).build();

// 26.2：BookMeta 本身可变，直接赋值
BookMeta meta = (BookMeta) item.getItemMeta();
meta.title(Component.text("指南"));
meta.pages(List.of(Component.text("第一页"), Component.text("第二页")));
item.setItemMeta(meta);

// 需要 Adventure 的 Book 对象时（如 openBook）
net.kyori.adventure.inventory.Book book = meta.asBook();
```

破坏性变更之三为方块怪的继承关系调整。`MagmaCube` 不再继承 `Slime`，两者均改为实现 `AbstractCubeMob`。`SlimeSplitEvent#getEntity()` 的返回类型随之变更，构成二进制破坏：在 26.1 编译的插件运行于 26.2 时将抛出 `NoSuchMethodError`，必须重新编译。

```java
// 原实现
if (entity instanceof Slime slime) { ... }

// 26.2 实现
if (entity instanceof AbstractCubeMob cube) {
    cube.setSize(1);   // 统一处理史莱姆、岩浆怪与硫方块
}
```

### 4.3 Folia 的 `folia-supported` 字段语义

Folia 是 PaperMC 的区域化多线程分支，将世界划分为多个独立区域并行 tick。该字段的性质为加载开关，而非兼容性声明。

依据 Folia 官方 README，未被显式标记的插件不会加载：

```yaml
folia-supported: true
```

未声明该字段时，插件不会出现在服务端插件列表中，不产生警告，也不降级运行。因此从 Paper 迁移至 Folia 时，多数插件将不再加载。Folia 官方对未修改的 Paper 插件给出的兼容性预期为 0。

文档同时包含部署相关信息：Folia 建议配置至少 16 个物理核心；线程分配参考值为 netty IO 每 200–300 名玩家约 4 线程、区块系统 IO 约 3 线程、世界预生成后区块 worker 约 2 线程；剩余核心可分配至 tick 线程（全局配置项 `threaded-regions.threads`），但线程总占用不应超过 CPU 核心数的 80%。此外，计分板 API、运行时世界创建与卸载、传送门与玩家重生相关 API 在 Folia 上不可用，`Entity#teleport` 永久不可用，应改用 `teleportAsync`。

## 五、安装与加载

该技能包在 dsh **0.1.5-rc.2** 下完成验证。运行环境要求 Node.js 22 及以上。

```bash
# 确认 Node 版本
node -v

# 获取技能包
git clone https://gitee.com/IYeaSakura/PaperMC-Dev-Skills.git

# 部署至用户级技能目录（对应扫描 rank 400）
mv PaperMC-Dev-Skills ~/.dsh/skills/Minecraft-Paper-Dev-Skills

# 校验目录结构
ls ~/.dsh/skills/Minecraft-Paper-Dev-Skills
# LICENSE  README.md  README_zh.md  SKILL.md  references
```

启动 dsh Web 界面：

```bash
npx @deepseek-ai/dsh web
```

界面地址为 `http://127.0.0.1:3080`。工作区应指定为插件项目所在目录，Agent 的写入权限限定于该目录范围内。

加载完成后，技能以 `minecraft-paper-dev-skills` 名称出现在会话技能目录中。dsh 会向模型下发该目录，并提示模型先加载技能再执行任务。加载方式有两种：

- 模型自行调用 `skill` 工具，以精确名称加载，获得完整指令正文；
- 用户在输入中使用 `/minecraft-paper-dev-skills` 显式调用，指令直接注入当前轮次，无需模型选择。

技能包内容为独立 Markdown 文档，不依赖 dsh 也可直接作为技术文档阅读。

## 六、平台覆盖

除 Paper 外，文档覆盖 Folia 与 Purpur 两个分支。

| 项目 | Paper | Folia | Purpur |
|---|---|---|---|
| 线程模型 | 单主线程 | 无主线程，按区域并行 | 单主线程 |
| 最新 26.2 构建 | 129-stable | 7-beta | 2633-stable |
| 最新 26.3 构建 | 133-alpha | 无 | 2642-experimental |
| 预发布通道命名 | alpha | beta | experimental |
| 插件声明要求 | 无 | `folia-supported: true` | 无 |
| 未修改 Paper 插件可用性 | 可用 | 基本不可用 | 可用，行为一致 |

需要区分「Minecraft 已发布」与「Paper 已稳定」两件事。Minecraft 26.3（Wilderness Bound）已于 2026-09-15 发布，最低 Java 版本仍为 25；但 Paper 26.3 停留在 alpha、Purpur 26.3 属于 experimental、Folia 尚无 26.3 构建，因此文档以 26.2 为生产目标，并单独说明 26.3 中插件可观察到的变化，例如新的 `org.bukkit.entity.Cushion` 实体类型与新增 `Material` 常量。

三个平台的依赖坐标不同：

```kotlin
compileOnly("io.papermc.paper:paper-api:26.2.build.129-stable")        // Paper
compileOnly("dev.folia:folia-api:26.2.build.7-beta")                  // Folia，groupId 为 dev.folia
compileOnly("org.purpurmc.purpur:purpur-api:26.2.build.2633-stable")  // Purpur，仓库为 repo.purpurmc.org
```

文档给出了单 JAR 跨平台方案：统一使用 Paper 提供的四个调度器（`getGlobalRegionScheduler`、`getRegionScheduler`、`getAsyncScheduler`、`Entity#getScheduler`）与 `teleportAsync`，编译目标保持为 `paper-api`。上述调度器在 Paper 上同样可用，执行时投递至主线程；在 Folia 上投递至对应区域线程。

Purpur 的注意事项为反向兼容问题。依据其官方 FAQ，`purpur.yml` 保持默认时行为与 Paper 完全一致；但配置项启用后，部分插件的原版假设将失效。文档列出了配置项与插件类型的对应关系，例如启用 `ridable` 会影响处理坐骑与载具的插件，启用 `clamp-attributes` 会影响读取 `AttributeInstance#getBaseValue()` 进行计算的插件。

## 七、版本时效性维护

文档中所有版本号核实于 2026-09-28。考虑到 Paper 的版本迭代周期，文档提供了版本核验命令：

```bash
# Paper：查询最新 stable 构建
curl -s https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml \
  | grep -o '<version>26\.[0-9.]*build\.[0-9]*-stable</version>' | tail -n 3

# Folia
curl -s https://repo.papermc.io/repository/maven-public/dev/folia/folia-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3

# Purpur
curl -s https://repo.purpurmc.org/snapshots/org/purpurmc/purpur/purpur-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3
```

需注意 Paper 的 Maven 元数据中 `<latest>` 与 `<release>` 两个标签当前均指向 `26.3-pre-2.build.0-alpha`。该值不仅属于 alpha 通道，而且并非最新的 alpha 构建（最新为 `26.3.build.133-alpha`）；元数据中的版本列表也并非按发布时间排列，`26.3-pre-2.build.0-alpha` 排在列表末尾。若依赖工具直接读取这两个标签或取列表末项进行升级，都会引入预发布版本，应改为解析版本列表并按构建号筛选 `-stable` 后缀。

同样的缺陷出现在另两个分支上：`folia-api` 的 `<latest>` 与 `<release>` 均为 `26.2.build.7-beta`，原因是 Folia 从未发布 26.2 的 stable 构建；Purpur 的 `metadata.latest` 为 `26.3.build.2642-experimental`，其最新构建属于 experimental 通道，而当前稳定目标仍为 `26.2.build.2633-stable`。

Javadoc 同样不能当作构建号来源。26.2 的 Javadoc 与构建同步（`26.2.build.129-stable`），但 26.3 的 Javadoc 在同期仍停留在 `26.3.build.49-alpha`，落后实际构建八十余个版本。从页面标题读取当前版本，会得到过时的结论。

版本核验的价值不在于数字本身，而在于数字背后的语义变更。最近一次核对中，Paper 在 26.3 提交中将整套 Bukkit 命令接口标记为 `@ApiStatus.Obsolete(since = "26.3")`，涵盖 `CommandExecutor`、`TabCompleter`、`PluginCommand`、`CommandMap` 与 `JavaPlugin#onCommand`，并在注解中给出两条替代路径：`BasicCommand` 与 Brigadier。该变更不产生编译警告，也不会让现有插件停止加载，但它是方向性的。此类信息不会出现在任何单一页面上，只能通过追踪上游提交获得，而这正是技能包需要持续维护的部分。

## 八、使用建议与改编

**提问时明确版本与平台。** 文档中部分内容按平台分别编写。指明平台后，模型将读取对应的 reference 文件。例如"Paper 26.2 环境下使用 Bukkit.getScheduler 编写的定时任务能否运行于 Folia"，比仅提问"Folia 兼容性"可获得更准确的结论。

**应用变更前要求说明依据。** 文档中每条结论均可追溯至来源。可要求模型说明判断依据后再执行修改。

**改编为其他领域。** 该技能包的结构可直接复用于其他技术领域，核心工作有三项：将描述压缩至 500 字符内并前置关键能力；将低频细节拆分至 `references/`；为每条结论标注核实来源与日期。README 的"改编为其他领域"一节提供了完整的七步清单。
