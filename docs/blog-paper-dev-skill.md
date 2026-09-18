# 做了个 Paper 插件开发 Skill：把 4600 行避坑文档装进 DeepSeek Harness，问它问题不用再翻源码

维护 Paper 插件的人大概都有过这种体验：想确认一个字段该填什么，得先翻官方 Javadoc，再翻 GitHub 上的源码，最后去 Maven 元数据里看构建号，一圈下来十几分钟就为了确认一个值。

我今年在这上面栽过几次，索性把核实过的结论整理成文档，做成了一个能被 AI 助手直接读取的知识包。现在问它 `api-version` 该填什么、床的 PDC 数据怎么迁移、插件能不能跑 Folia，都是直接给答案，不用再去查。

这个 Skill 叫 **Minecraft-Paper-Dev-Skills**，仓库在 Gitee：`gitee.com/IYeaSakura/PaperMC-Dev-Skills`。

## 它解决的是哪类问题

举几个我实际问过的例子，这些答案都不是模型自己"想"出来的，是文档里固化好的：

| 你问 | 通用 AI 常见回答 | 这个 Skill 的回答 |
|---|---|---|
| Paper 26.2 的 `api-version` 填什么 | `26.1.2` 或 `1.21` | `'26.2'`，并解释为什么 `26.1.2` 语义上比 `26.2` 小 |
| 我的插件能跑 Folia 吗 | 需要适配 | 先看有没有 `folia-supported: true`；没有的话 Folia 压根不会加载 |
| 26.2 升级要改什么 | 泛泛说"注意 API 变更" | 列 9 项具体弃用项 + 3 个硬破坏，附迁移代码 |
| 当前最新稳定构建号 | 编一个 | 124-stable，并告诉你怎么自己 `curl` 确认 |

差别在于**确定性**。模型对 Minecraft 插件的版本细节记忆很乱，因为这套版本体系 2026 年才换，语料里全是旧的 `1.21` 经验。文档把这些事实固化了，模型就不用猜。

## 怎么装

需要 Node.js 22+。Skill 是给 DeepSeek Harness（dsh）用的，dsh 是 DeepSeek 开源的本地 Agent 运行框架，读写文件需要你逐次授权。

```bash
# 确认 Node 版本
node -v

# 拉仓库
git clone https://gitee.com/IYeaSakura/PaperMC-Dev-Skills.git

# 放进用户级 Skill 目录
mv PaperMC-Dev-Skills ~/.dsh/skills/Minecraft-Paper-Dev-Skills

# 确认结构
ls ~/.dsh/skills/Minecraft-Paper-Dev-Skills
# LICENSE  README.md  README_zh.md  SKILL.md  references
```

启动：

```bash
npx @deepseek-ai/dsh web
```

浏览器打开 `http://127.0.0.1:3080`，选一个工作区（就选你放插件项目那个目录，AI 只能动这个目录里的东西），然后直接问就行。

装好之后它会以 `minecraft-paper-dev-skills` 出现在会话的 Skill 列表里。

## 里面装了什么

整个包 4600 行（含双语 README），结构是这样：

```
Minecraft-Paper-Dev-Skills/
├── SKILL.md                     401 行   ← 常驻索引：速查表、选型矩阵、检查清单
├── references/
│   ├── api-patterns.md          952 行   ← 事件、命令、调度器、GUI、物品、实体
│   ├── version-matrix.md        643 行   ← 版本矩阵、api-version 规则、迁移指南
│   ├── project-setup.md         408 行   ← Maven/Gradle 模板、三平台依赖坐标
│   ├── data-storage.md          402 行   ← SQLite、MySQL、HikariCP、Caffeine
│   ├── plugin-yml.md            328 行   ← plugin.yml / paper-plugin.yml 字段规范
│   ├── purpur.md                267 行   ← Purpur 独有 API 与配置影响
│   └── folia.md                 231 行   ← Folia 线程模型与损坏 API 清单
├── README.md / README_zh.md     494 + 494 行
└── LICENSE
```

为什么拆成这么多文件，而不是写成一个大的 `SKILL.md`？因为 Skill 的加载机制是分层的：`SKILL.md` 会作为索引常驻上下文，`references/` 里的文件只在模型真正需要时才读取。如果 4000 行全塞进索引，每个会话都要吃满，反而变慢变贵。

所以 `SKILL.md` 里只放四类东西：版本事实速查、三平台选型矩阵、快速开始骨架、性能与安全检查清单。剩下的按需展开。

**就算你不用 dsh，这些 Markdown 直接当文档看也没问题**，`references/` 里每个文件都能独立阅读。

## 挑三个内容给你看看质量

判断一份文档值不值得用，看它敢不敢给确定结论。挑三个实际内容：

### 一、`api-version`

这是最容易错的字段。Paper 解析它的代码在 `ApiVersion.java`：

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

按 `major.minor.patch` 三段解析，所以 `26.1.2` 是 major=26、minor=1、patch=2，**比 `26.2` 小**，它不是 `26.2` 的补丁版。

正确值去 Paper 仓库自己的 `gradle.properties` 看：

```properties
mcVersion=26.2
apiVersion=26.2   # the current API version for use in (paper-)plugin.yml files
channel=STABLE
```

**26.2 服务器写 `'26.2'`。** 想同时兼容 26.1 和 26.2 就写 `'26.1'`，前提是没用到 26.2 新增的 API。

### 二、26.2 的三个硬破坏

文档里 26.1 → 26.2 的变更分了两类：9 项弃用（有替代方案，能编译但该改）和 3 项硬破坏（直接编不过或运行时报错）。

硬破坏之一是床。26.2 里床变回普通方块，**不能再存 `PersistentDataContainer`**，`org.bukkit.block.Bed` 被标弃用且 `setColor()` 直接抛异常。之前存过数据的话得在数据修复时抢救：

```java
@EventHandler
public void onBlockEntityRemoved(AsyncServerDataFixerRemoveBlockEntityEvent event) {
    if (!event.getBlockEntityType().equals(Key.key("minecraft", "bed"))) return;

    PersistentDataContainerView pdc = event.getPersistentDataContainerView();
    String myData = pdc.get(new NamespacedKey(this, "my_key"), PersistentDataType.STRING);
    if (myData == null) return;

    // 这个事件在区块加载流程里触发，可能在 worker 线程
    // 重活别放这，丢给自己的线程池
    getServer().getAsyncScheduler().runNow(this, task ->
        getLogger().info("抢救出数据: " + myData));
}
```

另一个是 Adventure 5 造成的 `BookMeta` 变更，这是真实线上事故的来源：

```java
// 26.1 写法，26.2 编译不过
BookMeta built = meta.toBuilder().title(Component.text("指南")).build();

// 26.2：BookMeta 本身可变
BookMeta meta = (BookMeta) item.getItemMeta();
meta.title(Component.text("指南"));
meta.pages(List.of(Component.text("第一页"), Component.text("第二页")));
item.setItemMeta(meta);

// 需要 Adventure 的 Book 对象时
net.kyori.adventure.inventory.Book book = meta.asBook();
```

第三个是方块怪。`MagmaCube` 不再继承 `Slime`，两者都改成实现 `AbstractCubeMob`。`SlimeSplitEvent#getEntity()` 的返回类型变了，这是二进制破坏，26.1 编译的插件跑 26.2 会 `NoSuchMethodError`，必须重编。

```java
// 老代码
if (entity instanceof Slime slime) { ... }

// 26.2
if (entity instanceof AbstractCubeMob cube) {
    cube.setSize(1);   // 史莱姆、岩浆怪、硫方块统一处理
}
```

### 三、Folia 的 `folia-supported` 是开关不是声明

这条我一开始也理解错了，以为是个"我兼容 Folia"的标记。

Folia 是 PaperMC 的区域化多线程分支，把世界切成多个区域并行 tick。它的 README 写得很直白：**只有显式标记过的插件才会被加载**。也就是说没有这一行，插件列表里根本不会出现，不是警告也不是降级运行：

```yaml
folia-supported: true
```

所以从 Paper 换到 Folia 会发生什么就很清楚了：大部分插件"消失"。Folia 官方对未修改的 Paper 插件给出的兼容期望是 0。

文档里还有服主关心的部分：Folia 建议至少 16 个物理核心，线程分配上 netty IO 每 200–300 人约 4 线程、区块 IO 约 3、预生成过世界的话区块 worker 约 2，剩下的给 tick 线程但总占用别超 CPU 核数的 80%。另外计分板 API、运行时创建/卸载世界、传送门和重生相关 API 在 Folia 上都是坏的，`Entity#teleport` 永久不可用只能 `teleportAsync`。

## 覆盖的三个平台

Paper 之外还覆盖了 Folia 和 Purpur，因为现在开服选型绕不开这两个：

| 项目 | Paper | Folia | Purpur |
|---|---|---|---|
| 线程模型 | 单主线程 | 无主线程，按区域并行 | 单主线程 |
| 最新 26.2 构建 | 124-stable | 7-beta | 2633-stable |
| 预发布通道名 | alpha | beta | experimental |
| 插件需要额外声明 | 否 | `folia-supported: true` | 否 |
| 普通 Paper 插件能否加载 | 可以 | 基本不行 | 可以，行为不变 |

依赖坐标也是三套：

```kotlin
compileOnly("io.papermc.paper:paper-api:26.2.build.124-stable")            // Paper
compileOnly("dev.folia:folia-api:26.2.build.7-beta")                      // Folia，group 不是 io.papermc.paper
compileOnly("org.purpurmc.purpur:purpur-api:26.2.build.2633-stable")      // Purpur，仓库在 repo.purpurmc.org
```

文档里给了一个跨平台策略：**只要全程用 Paper 的四个调度器（`getGlobalRegionScheduler`、`getRegionScheduler`、`getAsyncScheduler`、`Entity#getScheduler`）加上 `teleportAsync`，一份 JAR 就能同时跑在三个平台上**，编译目标仍然是 `paper-api`。这四套调度器 Paper 上也有，行为是投递到主线程。

Purpur 那边要注意的是反向问题：它默认行为和 Paper 完全一样（官方 FAQ 明确说了，`purpur.yml` 不改就是 Paper），但**服主把开关打开之后**，某些插件假设就会失效。文档列了对应关系，比如开 `ridable` 会影响处理坐骑和载具的插件，开 `clamp-attributes` 会影响读 `AttributeInstance#getBaseValue()` 做计算的插件。所以插件报 bug 时先问一句服主 `purpur.yml` 改了什么。

## 文档会过期，所以留了复查方法

上面所有版本号核实于 **2026-09-17**。Paper 几个月一个版本，所以我把核对方法也写进文档了，你可以自己确认：

```bash
# Paper：找最新 stable
curl -s https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml \
  | grep -o '<version>26\.[0-9.]*build\.[0-9]*-stable</version>' | tail -n 3

# Folia
curl -s https://repo.papermc.io/repository/maven-public/dev/folia/folia-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3

# Purpur
curl -s https://repo.purpurmc.org/snapshots/org/purpurmc/purpur/purpur-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3
```

顺便说个坑：Paper 那份 Maven 元数据里，`<latest>` 和 `<release>` 两个标签当前都指向 `26.3-pre-2.build.0-alpha`，是 **alpha 版**，不代表最新稳定版。用工具自动读这两个标签升级依赖会把预发布版拉进来，得自己筛 `-stable`。

## 两个使用建议

**问的时候带上具体版本和场景。** 比如"Paper 26.2 上，我用 Bukkit.getScheduler 写的定时任务能跑 Folia 吗"比"Folia 兼容性"能得到更准的答案。文档里有些内容是分平台写的，说清楚平台它会直接读对应的 reference。

**改动代码前先让它解释依据。** 文档里每条结论都能追到来源（源码、官方 Javadoc、Maven 元数据），你可以让它说明为什么这么判，再决定要不要照做。这是我做这份东西的原则：**宁可标注核实日期和来源，也不给一个听起来合理但没法验证的答案。**

---

仓库地址：`https://gitee.com/IYeaSakura/PaperMC-Dev-Skills`（GitHub 镜像 `IYeaSakura/PaperMC-Dev-Skills`）

有发现结论和当前版本对不上的，欢迎评论区指出，我会更新进去。
