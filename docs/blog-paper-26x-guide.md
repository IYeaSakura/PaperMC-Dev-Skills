# Paper 26.2 插件开发避坑指南：版本号、api-version 和三平台分支

Minecraft Java 版的版本号在 2026 年换了一套体系，从 `1.21.4` 变成了 `26.2` 这种写法。这个改动看起来只是数字变了，实际影响比想象中大：插件元数据字段的取值、Maven 依赖坐标、发布通道命名、甚至世界存档目录结构，全套跟着改了。

我平时维护一套 Paper 插件，今年在这上面栽过几次，后来干脆把核实过的结论整理成了一份文档，并且做成了一个能被 AI 助手直接读取的知识包。现在遇到版本相关问题基本不用再去翻源码。

这篇把里头最实用的部分抽出来，都是对照 Paper 源码、官方 Javadoc 和 Maven 元数据核实过的。顺带说一下怎么让 AI 助手也用上这份东西，省得它给你编版本号。

## 你现在写的 `api-version` 很可能是错的

先说最常见的一个坑。

网上的教程和不少插件的 `plugin.yml` 里，`api-version` 现在有几种写法：`1.21`、`26.1.2`、`26.2`。其中至少有一种是错的，而且错得不容易发现。

Paper 解析这个字段的代码在 `org.bukkit.craftbukkit.util.ApiVersion`：

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

它是按 `major.minor.patch` 三段来解析的。于是：

| 你写的 | 解析结果 | 实际含义 |
|---|---|---|
| `26.2` | major=26, minor=2, patch=0 | 26.2 |
| `26.1.2` | major=26, minor=1, patch=2 | 26.1.2 |
| `1.21` | major=1, minor=21, patch=0 | 1.21 |

关键在于，**`26.1.2` 并不是 `26.2` 的补丁版本**。按这个解析规则，`26.1.2` 比 `26.2` 小。你如果在 26.2 服务器上写 `api-version: '26.1.2'`，服务器会放行。但如果你用了 26.2 才有的 API，这个声明就是错的，而且它掩盖了问题，因为服务器本该拒绝这种越界调用。

那正确值是什么？去翻 Paper 仓库自己的 `gradle.properties`，写得很清楚：

```properties
mcVersion=26.2
apiVersion=26.2   # the current API version for use in (paper-)plugin.yml files
channel=STABLE
```

**26.2 服务器，`api-version` 就写 `'26.2'`。**

再补两条规则：

- 服务器配置里的 `settings.minimum-api`（默认 `none`）是下限。你的 `api-version` 比它低会被拒绝加载。
- 想同时兼容 26.1 和 26.2 服务器，就写 `'26.1'`，前提是你确实没用到 26.2 新增的 API。声明旧版本意味着服务器会按旧 API 对待你。

顺便提一句，Java 版本也是硬性的。26.x 全线要求 **Java 25**，用 21 编译出来的插件在 26.x 服务器上直接 `UnsupportedClassVersionError`。

## 26.1 到 26.2 之间坏了哪些东西

如果是从 26.1 升级到 26.2，下面这些改动值得逐个对照检查。

**床不再是方块实体。** 这条影响最大。26.2 里床变回了普通方块，意味着它**不能再存 `PersistentDataContainer`**。`org.bukkit.block.Bed` 被标了 `@Deprecated(forRemoval=true)`，而且 `setColor()` 现在直接抛 `UnsupportedOperationException`。

如果你之前往床的 PDC 里存过数据，需要在数据修复时抢救出来：

```java
@EventHandler
public void onBlockEntityRemoved(AsyncServerDataFixerRemoveBlockEntityEvent event) {
    if (!event.getBlockEntityType().equals(Key.key("minecraft", "bed"))) return;

    PersistentDataContainerView pdc = event.getPersistentDataContainerView();
    String myData = pdc.get(new NamespacedKey(this, "my_key"), PersistentDataType.STRING);
    if (myData == null) return;

    // 注意：这个事件在区块加载过程中触发，可能在 worker 线程
    // 不要把重活放在这里，丢给自己的线程池
    getServer().getAsyncScheduler().runNow(this, task ->
        getLogger().info("抢救出数据: " + myData));
}
```

这个事件还有个容易忽略的地方：名字带 `Async`，实际是在区块加载流程里触发的，可能在 worker 线程也可能在主线程。Javadoc 明确说重活和阻塞操作会拖慢服务器，甚至卡住主线程（因为主线程可能在等这些 worker）。

**Adventure 5 让书本相关代码编译不过。** Paper 26.2 升级到了 Adventure 5，`BookMeta` 不再实现 Adventure 的 `Book` 接口，老的 builder 也被删了：

```java
// 26.1 的老写法，26.2 编译不过
BookMeta built = meta.toBuilder()
    .title(Component.text("指南"))
    .pages(List.of(Component.text("第一页")))
    .build();

// 26.2：BookMeta 本身可变，直接改
BookMeta meta = (BookMeta) item.getItemMeta();
meta.title(Component.text("指南"));
meta.pages(List.of(Component.text("第一页"), Component.text("第二页")));
item.setItemMeta(meta);

// 需要 Adventure 的 Book 对象（比如 openBook）时
net.kyori.adventure.inventory.Book book = meta.asBook();
```

这是个真实的线上事故来源。只调用过继承来的 `Book` 方法的插件，在 26.1 上编译正常，到了 26.2 运行时就 `NoSuchMethodError`。

**方块怪不能再用 `Slime` 判断了。** 26.2 里 `MagmaCube` 不再继承 `Slime`，两者都改成实现 `AbstractCubeMob`。这是个二进制破坏：`SlimeSplitEvent#getEntity()` 的返回类型变了，在 26.1 编译的插件跑到 26.2 上会 `NoSuchMethodError`，必须重新编译。

```java
// 老代码
if (entity instanceof Slime slime) { ... }

// 26.2
if (entity instanceof AbstractCubeMob cube) {
    cube.setSize(1);   // 史莱姆、岩浆怪、硫方块统一处理
}
```

顺便说，这个改动当时把 `RegionAccessor#spawn` 也搞坏过一阵，生成史莱姆会抛 `IllegalArgumentException`。用 26.2 的 build #30 之后的版本没这个问题。

**其他值得扫一眼的弃用项：**

| 弃用项 | 替代方案 |
|---|---|
| `PointedDripstone` 方块数据 | `Speleothem` |
| `World#setSpawnFlags()`、`getAllowAnimals()` | 直接删掉，原版没有动物生成开关了 |
| `Vex#getSummoner()` | `Vex#getOwner()` |
| `BlockSoundGroup`（destroystokyo） | `org.bukkit.SoundGroup` |
| `TargetBlockInfo` | `RayTraceResult` + `FluidCollisionMode` |
| `BukkitBrigadierCommand`、`PaperBrigadier` | `Commands` + `LifecycleEvents.COMMANDS` |
| `TeleportFlag.EntityState` 那几个 flag | 现在默认就是原版行为，不用传了 |
| 整个 `org.bukkit.conversations` 包 | `AsyncChatEvent` 或 `Dialog` |
| `co.aikar.timings.*` | spark |

Timings 那条对服主也有意义：以后看性能问题用 spark 的 `/spark profiler`，别再用 `/timings` 了。

## 服务器版本和构建号怎么读

Paper 现在的构件版本格式是：

```
<minecraft-version>.build.<构建号>-<通道>
       26.2        .build.  124    -stable
```

通道的官方定义是这样：

| 通道 | 含义 |
|---|---|
| `alpha` | 以前叫 experimental，容易出错，官方不提供支持 |
| `beta` | 中间状态，不是完全不稳定，但还有没做完的部分 |
| `stable` | 以前叫 default，重要修复持续推送到这里 |

**生产环境只用 stable。** 官方原话大意是"你应该始终使用 stable 构建，实验性构建容易出错且不受支持"。

查当前版本和构建号：

```bash
# Paper：列出 26.2 的所有构建及通道
curl -s https://fill.papermc.io/v3/projects/paper/versions/26.2/builds

# Maven 元数据里找最新的 stable
curl -s https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml \
  | grep -o '<version>26\.[0-9.]*build\.[0-9]*-stable</version>' | tail -n 3
```

这里有个坑要说一下。上面那份 Maven 元数据里，`<latest>` 和 `<release>` 两个标签当前都指向 `26.3-pre-2.build.0-alpha`，是个 **alpha 版**。它们不代表最新稳定版。如果你用工具自动读这两个标签来升级依赖，会把预发布版拉进来。得自己解析版本列表筛 `-stable`。

另外旧版 `https://api.papermc.io/v2` 的下载接口已经下线了（返回 HTTP 410），改用 `fill.papermc.io/v3`。

至于 26.3：Minecraft 26.3 在 2026-09-15 已经正式发布，但 **Paper 26.3 目前只有 alpha 构建**。生产服务器别急。

## 服主需要知道的两个分支

### Folia：不是"线程更多的 Paper"

Folia 是 PaperMC 官方的区域化多线程分支，把世界切成多个独立区域并行 tick。听起来很美好，但它是另一套运行模型，对服务端管理员的影响很直接：

**未声明兼容的插件根本不会加载。** 必须在插件的 `plugin.yml` 里有这一行：

```yaml
folia-supported: true
```

不加的话，插件列表里压根不会出现，也不是报错禁用，就是不加载。所以如果你从 Paper 换到 Folia，大部分插件会"消失"。Folia 官方 README 对未修改的 Paper 插件给出的兼容期望是 0。

**硬件要求不低。** 官方建议至少 16 个物理核心（不是线程），并且更适合玩家分散的服（空岛、生存服），人多且挤在一起的效果不明显。

**那 tick 线程数怎么配**，官方给的是基于当年 330 人峰值测试的粗略估计：netty IO 每 200–300 人约 4 线程，区块系统 IO 每 200–300 人约 3 线程，预生成过世界的话区块 worker 每 200–300 人约 2 线程。剩下分配给 tick 线程（全局配置里的 `threaded-regions.threads`），但**所有线程加起来不要超过 CPU 核数的 80%**，因为插件和服务器自己还会起一些你控制不了的线程。

还有几个功能在 Folia 上是坏的，迁移前要有心理准备：计分板 API 整体不可用、运行时创建/卸载世界不可用、传送门和玩家重生相关 API 有问题、`Entity#teleport` 永久不可用（只能 `teleportAsync`）。

### Purpur：默认行为和 Paper 一样

Purpur 是 Paper 的直接替换版，加了一堆可配置的玩法功能。对服主来说最好的一点是官方 FAQ 里的这句话：

> 如果 `purpur.yml` 里什么都不改，跑这个 JAR 和跑 Paper 没有任何区别。

所有改动默认关闭。所以换过去基本没有风险，不会因为换了服务端就出问题。

但对插件作者来说，风险在于**服主把某些开关打开之后**：

| 打开的配置 | 可能影响的插件 |
|---|---|
| 生物的 `ridable` / `controllable` | 处理坐骑、载具、`PlayerInteractEntityEvent` 的插件 |
| 方块行为修改（`farmland.disable-trampling`、`leaves.instant-decay`、`lava.speed` 等） | 依赖原版方块时序的逻辑 |
| `clamp-attributes`、各生物的 `attributes.*`、`limit-armor` | 读 `AttributeInstance#getBaseValue()` 做计算的插件 |
| 额外命令（`/tpsbar`、`/rambar`、`/compass`、`/afk` 等） | 命令名可能冲突 |

所以插件报 bug 的时候，如果服主用的是 Purpur，值得先问一句 `purpur.yml` 改了哪些。反过来说，插件里也别硬编码原版数值。

## 26.1 那次世界存储改动，升级前必须备份

这条给服主。

26.1 改了世界的存储结构。以前除了 `level.dat`，维度数据散在服务器根目录（`world/`、`world_nether/`、`world_the_end/`），现在统一收到 `world/dimensions/` 下面：

```
world/
├── data/minecraft/…
├── datapacks/
├── dimensions/
│   └── minecraft/
│       ├── overworld/
│       │   ├── data/minecraft/     # weather.dat 等
│       │   ├── data/paper/         # persistent_data_container.dat 等
│       │   ├── entities/  poi/  region/
│       │   └── paper-world.yml
│       ├── the_nether/
│       └── the_end/
├── players/
└── level.dat
```

**升级后不能回退。** Paper 官方公告里明确写了升级到 26.1 之后无法降级。所以升级前务必备份，这不是客套话。

顺带影响：如果你的插件或运维脚本硬编码了 `world/persistent_data_container.dat`、`world/paper-world.yml`、`world_nether/` 这类路径，都得改。

## 附：怎么让 AI 助手也别再编版本号

上面这些内容我整理成了一份结构化文档，并且做成了可以被 DeepSeek Harness 直接读取的技能包。因为它把可靠的版本事实固化了，问它 Paper 26.2 的问题不会得到 `1.26.2` 这种答案。

DeepSeek Harness（简称 dsh）是 DeepSeek 开源的 Agent 运行框架，本地跑，读写文件需要你授权。

**安装：**

```bash
# 先装 Node.js 22+
node -v

git clone https://gitee.com/IYeaSakura/PaperMC-Dev-Skills.git
mv PaperMC-Dev-Skills ~/.dsh/skills/Minecraft-Paper-Dev-Skills

# 确认目录结构
ls ~/.dsh/skills/Minecraft-Paper-Dev-Skills
# LICENSE  README.md  README_zh.md  SKILL.md  references
```

**启动：**

```bash
npx @deepseek-ai/dsh web
```

浏览器会打开 `http://127.0.0.1:3080`。选一个工作区（就是你放插件项目的目录，注意 AI 只能动这个目录里的东西），然后直接问，比如：

- `Paper 26.2 的 plugin.yml 里 api-version 应该写什么`
- `我的插件用了 Bukkit.getScheduler()，能跑在 Folia 上吗`
- `床的 PDC 数据在 26.2 丢了，怎么写迁移代码`
- `帮我把 26.1 的 BookMeta 代码改成 26.2 能编译的`

装好之后它会出现在会话的技能列表里，名字是 `minecraft-paper-dev-skills`。

**文档内容：**

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 版本事实速查、三平台选型矩阵、快速开始、检查清单 |
| `references/version-matrix.md` | 完整版本矩阵、`api-version` 规则、迁移指南、26.x 破坏性变更清单 |
| `references/project-setup.md` | Maven/Gradle 配置模板、Java 25 toolchain、三平台依赖 |
| `references/plugin-yml.md` | `plugin.yml` 与 `paper-plugin.yml` 字段规范 |
| `references/api-patterns.md` | 事件、命令、调度器、GUI、物品、实体代码示例 |
| `references/folia.md` | Folia 线程模型、四个调度器、损坏 API 清单、迁移检查表 |
| `references/purpur.md` | Purpur 独有 API、`purpur.yml` 配置影响、权限 |
| `references/data-storage.md` | SQLite、MySQL、HikariCP、Caffeine 缓存模式 |

不想装 dsh 的话，这些 Markdown 直接当文档看也行，仓库里还有双语 README。

## 文档会过期，所以我把复查命令写进去了

上面所有版本号核实于 **2026-09-17**。Paper 的更新节奏是几个月一个版本，所以我把核对方法也留在文档里了：每个版本号都能追溯到来源（Paper 源码、官方 Javadoc、Maven 元数据），并且附了前面那几条 `curl` 命令，你可以自己确认当前构建号再决定要不要更新。

仓库地址：`https://gitee.com/IYeaSakura/PaperMC-Dev-Skills`（GitHub 镜像 `IYeaSakura/PaperMC-Dev-Skills`）

有发现文中结论和当前版本对不上的，欢迎评论区指出，我会更新进去。
