---
name: minecraft-paper-dev-skills
description: PaperMC 26.x (Minecraft Java 26.x) plugin development guide covering Paper, Folia and Purpur, targeting Paper 26.2 stable and Java 25. Use when the user needs to develop, code, debug, or maintain PaperMC server plugins. Covers Java 25, Maven/Gradle project setup, plugin.yml/paper-plugin.yml configuration, Bukkit/Paper API (events, commands, schedulers, GUI, items, entities, worlds), Adventure 5 / MiniMessage text, data persistence (SQLite/MySQL), performance optimization, security best practices, version compatibility, migration from 1.21.x or 26.1 to 26.2/26.3, Folia regionised multithreading (folia-supported flag, region/entity schedulers, thread ownership checks), and Purpur fork features (org.purpurmc.purpur API, purpur.yml options, permissions). Also applies when user asks about Bukkit/Spigot/Paper/Folia/Purpur plugin development, Minecraft server plugins, or migrating plugins to newer Paper versions.
---

# PaperMC Plugin Development Skill

Comprehensive guide for developing plugins for **Paper and its major forks (Folia, Purpur)** on 26.x, using Java 25 and Maven/Gradle.

**Target versions (as of 2026-09-17):** Paper **26.2** = latest **stable** (Minecraft Java 26.2), Paper **26.3** = **alpha only**, Paper **26.1.x** = previous stable line. Always code against the latest stable line (26.2 today) unless the user explicitly targets another.

## Server Flavors: Paper, Folia, Purpur

Pick the target deliberately — the three differ substantially in what a plugin may assume.

| | **Paper** | **Folia** | **Purpur** |
|---|---|---|---|
| What it is | The base server | Paper fork adding **regionised multithreading** | Paper **drop-in replacement** with opt-in gameplay/config patches |
| Threading | One main thread | **No main thread**; one tick loop per region, ticking in parallel | One main thread (not a Folia fork) |
| Latest 26.2 build | `26.2.build.124-stable` | `26.2.build.7-beta` (**beta**) | `26.2.build.2633-stable` |
| Maven API coordinate | `io.papermc.paper:paper-api` | `dev.folia:folia-api` | `org.purpurmc.purpur:purpur-api` |
| Plugin opt-in flag | — | `folia-supported: true` required, else the plugin is **not loaded** | — |
| Vanilla plugins work? | Yes | **Almost none** — Folia's own README puts expectations at 0 | Yes, unchanged (features are off by default) |
| Own API surface | Bukkit + `io.papermc.paper` | `io.papermc.paper.threadedregions.scheduler`, `Bukkit#isOwnedByCurrentRegion` | `org.purpurmc.purpur.*` events/entity/language |
| Reference | this file + [references/api-patterns.md](references/api-patterns.md) | [references/folia.md](references/folia.md) | [references/purpur.md](references/purpur.md) |

**Practical guidance:**

- **Default to Paper.** Use only the four Paper schedulers (`getGlobalRegionScheduler`, `getRegionScheduler`, `getAsyncScheduler`, `Entity#getScheduler`) and `teleportAsync` instead of the legacy equivalents, and the same JAR will run on Paper **and** Folia.
- **Folia is not "Paper with more threads".** It refuses to load plugins that don't declare `folia-supported: true`, there is no main thread, and the scoreboard API, world load/unload, portals/respawn and `Entity#teleport` are broken. See [references/folia.md](references/folia.md) before targeting it.
- **Purpur needs no code changes** for ordinary Paper plugins: everything it adds is off unless an admin enables it in `purpur.yml`. The real risk is a plugin that assumes vanilla mechanics (rideable mobs, block behaviour, attribute values) on a server where those toggles are on. See [references/purpur.md](references/purpur.md).

## Version Facts

### Paper's new versioning scheme (since 26.1)

Paper replaced the old `1.21.8-R0.1-SNAPSHOT` Maven/artifact style with a scheme that carries the build and its maturity channel:

```
<minecraft-version>.build.<number>-<channel>
       26.2      .build.  124   -stable
```

| Channel | Meaning | Use for |
|---------|---------|---------|
| `-alpha` | Formerly "experimental"; prone to error, unsupported | Testing upcoming versions only |
| `-beta` | Middle point: not entirely unstable, but partially unfinished | Early plugin compatibility testing |
| `-stable` | Formerly "default"; fixes keep landing here | **Production and releases — the only channel to serve** |

(`recommended` exists in the downloads service but is unused by Paper; it is used by Velocity.)

- **Latest stable API:** `io.papermc.paper:paper-api:26.2.build.124-stable`
- **Latest published artifact:** `26.3-pre-2.build.0-alpha` (newer, but alpha — do not ship on it)
- Maven repository metadata lives at `https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml`; the build/channel list lives at `https://fill.papermc.io/v3/projects/paper`. The old `https://api.papermc.io/v2` downloads API is **sunset** and returns HTTP 410 — do not rely on it. Also note that in the Maven metadata `<latest>`/`<release>` both point at an **alpha** build, so parse the version list for the newest `-stable` rather than trusting those tags.

### Other version facts

- **Minecraft Java versioning changed in 2026** to `year.drop.patch` (e.g. `26.2`), so Paper 26.2 is **not** 1.26.x and **not** 1.21.x. Do not translate old `1.21` numbers into `1.26`.
- **Minimum Java: 25.** Java 21 or lower fails at server start and at compile time (`UnsupportedClassVersionError`). This applies to the whole 26.x line.
- **Java target:** compile and run with Java 25 (`<release>25</release>` / toolchain 25). Java 25 is an LTS release, so it is the safe floor, not a bleeding-edge choice.
- **Hardfork:** since 1.21.4 (2024-12) Paper is independent from Spigot. Existing Spigot API methods still work, but new plugins should compile against Paper-API.
- **Mojang mappings:** Paper 26.x ships a Mojang-mapped runtime. NMS access uses Mojang class names (e.g. `net.minecraft.world.entity.player.Player`), and plugins that touch NMS should be built Mojang-mapped (see [references/version-matrix.md](references/version-matrix.md)).

### api-version in plugin.yml (most common mistake)

`api-version` takes the **Minecraft/Paper API version**, never the Paper build string:

```yaml
api-version: '26.2'     # CORRECT on Paper 26.2  (major.minor)
api-version: '26.1'     # older 26.x line, still 'major.minor'
```

Rules enforced by Paper (`org.bukkit.craftbukkit.util.ApiVersion`):

- The value must parse as `major.minor` or `major.minor.patch` with **numeric** parts, e.g. `1.20.5`, `1.21`, `26.2`, `26.1.2`.
- A plugin whose `api-version` is **newer** than the server's API version is refused: `InvalidPluginException: Unsupported API version …`.
- A plugin below the server's configured floor (`settings.minimum-api` in `bukkit.yml`, default `none`) is refused too.
- Omitting it makes Paper log `Legacy plugin … does not specify an api-version.` and, on old versions, triggers Legacy Material Support.
- **Never write the Paper build number** (`26.2.build.124-stable`) or a `1.26.x` value — both are invalid.

Pick the **lowest** `api-version` that supports every API you call. Declaring `26.2` when you only use pre-26.2 API prevents your plugin from loading on 26.1 servers; declaring `26.1` keeps 26.1 + 26.2 compatibility.

## Quick Start Workflow

### 1. Project Setup

Reference: [references/project-setup.md](references/project-setup.md) for complete Maven `pom.xml` and Gradle `build.gradle.kts` templates.

**Maven essentials:**
```xml
<properties>
    <maven.compiler.release>25</maven.compiler.release>
    <!-- Pin an exact -stable build; Paper labels Maven ranges "Discouraged".
         Gradle may instead use "26.2.build.+" — keep the literal `build` token. -->
    <paper.api.version>26.2.build.124-stable</paper.api.version>
</properties>

<repositories>
    <repository>
        <id>papermc</id>
        <url>https://repo.papermc.io/repository/maven-public/</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>io.papermc.paper</groupId>
        <artifactId>paper-api</artifactId>
        <version>${paper.api.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

### 2. plugin.yml (Required)

Place in `src/main/resources/plugin.yml`:

```yaml
name: YourPlugin
version: '${project.version}'
main: com.yourname.yourplugin.YourPlugin
api-version: '26.2'
description: Your plugin description
author: YourName

commands:
  yourcmd:
    description: Main command
    usage: /yourcmd <subcommand>
    permission: yourplugin.use

permissions:
  yourplugin.use:
    description: Basic permission
    default: true
  yourplugin.admin:
    description: Admin permission
    default: op
```

**Critical rules:**
- `name`: only letters, numbers, underscores, hyphens. No spaces.
- `api-version`: use `'26.2'` (or lower `26.x` you actually support). See the section above.
- `main`: must extend `org.bukkit.plugin.java.JavaPlugin`.

For `paper-plugin.yml` format (alternative): see [references/plugin-yml.md](references/plugin-yml.md).

### 3. Main Class

```java
package com.yourname.yourplugin;

import org.bukkit.plugin.java.JavaPlugin;

public class YourPlugin extends JavaPlugin {
    private static YourPlugin instance;

    @Override
    public void onEnable() {
        instance = this;
        saveDefaultConfig();
        getServer().getPluginManager().registerEvents(new YourListener(), this);
        getCommand("yourcmd").setExecutor(new YourCommand());
        getCommand("yourcmd").setTabCompleter(new YourTabCompleter());
        getLogger().info("YourPlugin enabled!");
    }

    @Override
    public void onDisable() {
        // 26.x: prefer the Paper schedulers so the plugin also runs on Folia
        getServer().getGlobalRegionScheduler().cancelTasks(this);
        getServer().getAsyncScheduler().cancelTasks(this);
        getLogger().info("YourPlugin disabled!");
    }

    public static YourPlugin getInstance() { return instance; }
}
```

**Initialization order (onEnable):**
1. saveDefaultConfig / reloadConfig
2. Database connection
3. Cache layer
4. Manager classes
5. Register event listeners
6. Register commands + tab completers (or `LifecycleEvents.COMMANDS`)
7. Start scheduled tasks

**Cleanup order (onDisable):**
1. Cancel all scheduled tasks
2. Save cache / flush data
3. Close database connections
4. Clean up resources

### 4. Build & Deploy

```bash
# Maven
mvn clean package
# Output: target/your-plugin-1.0.0-SNAPSHOT.jar

# Copy to server plugins/ folder and restart
```

## Key API Patterns

Reference: [references/api-patterns.md](references/api-patterns.md) for detailed examples.

### Event Listener

```java
public class PlayerListener implements Listener {
    @EventHandler
    public void onPlayerJoin(PlayerJoinEvent event) {
        Player player = event.getPlayer();
        player.sendMessage(Component.text("Welcome!"));
    }

    @EventHandler(priority = EventPriority.HIGH, ignoreCancelled = true)
    public void onBlockBreak(BlockBreakEvent event) {
        // High priority, skip if cancelled by another plugin
    }
}
```

Register: `getServer().getPluginManager().registerEvents(new PlayerListener(), this);`

### Command with Tab Completion

```java
// Main command executor with subcommand routing
public class MainCommand implements CommandExecutor {
    private final Map<String, SubCommand> subCommands = new HashMap<>();

    public MainCommand() {
        subCommands.put("reload", new ReloadSubCommand());
        subCommands.put("give", new GiveSubCommand());
    }

    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        if (args.length == 0) { sender.sendMessage("Usage: /cmd <sub>"); return true; }
        SubCommand sub = subCommands.get(args[0].toLowerCase());
        if (sub == null) { sender.sendMessage("Unknown command"); return true; }
        return sub.execute(sender, Arrays.copyOfRange(args, 1, args.length));
    }
}
```

For new code prefer the Paper Brigadier API (`LifecycleEvents.COMMANDS`), which is the supported command path on 26.x. See [references/api-patterns.md](references/api-patterns.md).

### Scheduler (Critical Thread Safety)

| Operation | Required Thread |
|-----------|----------------|
| Modify blocks | Owning region thread (main on Paper) |
| Operate entities | Owning region thread (main on Paper) |
| Player inventory | Owning region thread (main on Paper) |
| Database queries | Async |
| File I/O | Async |
| HTTP requests | Async |

Use the four Paper schedulers shown below — on Paper and Purpur they run on the main thread, on Folia on the owning region thread, so the same code is portable. **Folia additionally requires `folia-supported: true` in `plugin.yml`, has no main thread, and breaks scoreboards, world load/unload, portals/respawn and `Entity#teleport` (use `teleportAsync`).** Full details: [references/folia.md](references/folia.md).

```java
// Async task, then back to main thread
getServer().getAsyncScheduler().runNow(plugin, task -> {
    PlayerData data = database.load(uuid);
    getServer().getGlobalRegionScheduler().run(plugin, scheduledTask ->
        player.sendMessage(Component.text("Loaded: " + data.getCoins())));
});
```

`Bukkit.getScheduler()` still works on Paper (excluding Folia), but the Paper region/async/global schedulers are the forward-compatible choice on 26.x.

### GUI / Inventory

Create an `InventoryHolder` implementation, open with `player.openInventory(inv)`. Handle `InventoryClickEvent` and `InventoryDragEvent`, always `event.setCancelled(true)` for GUI clicks. Use `PersistentDataContainer` (or item data components) to tag GUI items with action identifiers.

### ItemStack with Custom Data

```java
ItemStack item = new ItemStack(Material.DIAMOND_SWORD);
ItemMeta meta = item.getItemMeta();
meta.displayName(Component.text("Legendary Sword"));
meta.lore(List.of(Component.text("Line 1"), Component.text("Line 2")));
meta.getPersistentDataContainer().set(
    new NamespacedKey(plugin, "item_id"), PersistentDataType.STRING, "legendary_sword"
);
item.setItemMeta(meta);
```

Use Adventure `Component`s (`displayName`, `lore`) rather than `setDisplayName(String)`/`setLore(List<String>)`, which are legacy.

## Data Persistence

Reference: [references/data-storage.md](references/data-storage.md) for complete SQLite/MySQL/Caffeine patterns.

**Rule of thumb:**
- Small server / simple data: SQLite (driver ships with the server, no extra dependency)
- Large server / concurrent access: MySQL + HikariCP connection pool
- Always use `PreparedStatement` (never string concatenation for SQL)
- Cache frequently accessed data with Caffeine
- Use UUID (never Player object) as Map keys to avoid memory leaks

## Performance & Security Checklist

### Performance
- [ ] Database operations run async
- [ ] Use HikariCP for MySQL
- [ ] Batch insert instead of loop-insert
- [ ] Throttle `PlayerMoveEvent` (check time delta or chunk boundary)
- [ ] Cache with Caffeine for read-heavy data
- [ ] Use UUID keys, not Player objects
- [ ] Use `StringBuilder` for string concatenation in loops
- [ ] Cancel tasks and close DB in `onDisable`
- [ ] Use `try-with-resources` for all closables
- [ ] Profile with **spark** (`/spark profiler`) — the old **Timings** API is terminally deprecated for removal

## Version Compatibility

Reference: [references/version-matrix.md](references/version-matrix.md) for full compatibility details.

| PaperMC | Minecraft | Min Java | api-version | Status / Notes |
|---------|-----------|----------|-------------|----------------|
| 1.20.5 – 1.20.6 | 1.20.5 – 1.20.6 | 21 | `1.20` | Mojang-mapped runtime |
| 1.21 – 1.21.11 | 1.21 – 1.21.11 | 21 | `1.21` | Hardfork from Spigot |
| 26.1.1 | 26.1.1 | **25** | `26.1` | Unsupported (support ended 2026-04-11, last build 29 alpha) |
| 26.1.2 | 26.1.2 | **25** | `26.1` / `26.1.2` | Supported; world storage format change |
| **26.2** | **26.2** | **25** | **`26.2`** | **Latest stable** (stable since build 83, 2026-07-26; latest build 124) — Adventure 5, beds are no longer block entities |
| 26.3 | 26.3 | 25 | `26.3` | **Paper alpha only** — Minecraft 26.3 released 2026-09-15, Paper still pre-release |

There is no Paper version literally named "26.1" — the 26.1 line is `26.1.1` and `26.1.2`, while `26.1` is a perfectly valid `api-version` meaning "needs at least the 26.1 API".

**Breaking changes to know on 26.x:**
- **World storage format changed in 26.1** — dimensions now live under `world/dimensions/…` and per-world `paper-world.yml` moved with them. Upgrading is irreversible (no downgrade). Back up worlds first.
- **World names are obsolete** — `WorldInfo#getName()` is obsolete; use the `getKey()` methods, since Paper is moving away from world names internally.
- **Beds are no longer block entities** (26.2). `org.bukkit.block.Bed` is deprecated; beds can no longer hold a `PersistentDataContainer`. Migrate stored data and listen to `AsyncServerDataFixerRemoveBlockEntityEvent`.
- **Adventure 5** (26.2): `BookMeta` no longer extends Adventure's `Book`; the old builder is gone — use `BookMeta`'s own page methods. Deprecated `ClickEvent`/`HoverEvent` usage was removed, `Component` implementations are sealed, and chat-signing/`MessageType` API is gone.
- **Obfuscated plugins are unsupported from 26.1** — the internal remapper was dropped, so `reobfJar` no longer applies; ship Mojang-mapped.
- **Per-world clocks** (26.1): a new `ClockTimeSkipEvent` is the parent of `TimeSkipEvent`, and with `time.affects-all-worlds` enabled **only the parent event fires** for most command/API actions.
- **Conversation API** (`org.bukkit.conversations`) is deprecated for removal — use `AsyncChatEvent` or `Dialog`.
- **Timings** is terminally deprecated — use spark.
- Enum-like keyed types (Biome, Art, PatternType, …) keep `valueOf`/`values` only for compatibility; use `Registry.get(NamespacedKey)` / `Registry.stream()`.
- Cube mobs share `AbstractCubeMob` (26.2); `MagmaCube` no longer extends `Slime` and `SlimeSplitEvent#getEntity` returns `AbstractCubeMob`.

**Paper vs Spigot after hardfork:**
- Existing Spigot API methods continue to work
- Paper may add new API not in Spigot
- Plugins compiled against Paper-API may not run on Spigot
- Paper market share is ~85–90%, making Paper-only plugins viable
- Compiling Mojang-mapped is the recommended default on 26.x and drops the Spigot runtime remap

**Fork compatibility at a glance:**
- A plugin using only Paper schedulers + `teleportAsync` runs on Paper, Purpur **and** Folia (once you add `folia-supported: true`).
- A plugin using `Bukkit.getScheduler()`, scoreboards, world load/unload or `Entity#teleport` runs on Paper and Purpur but **not** Folia.
- A plugin importing `org.purpurmc.purpur.*` runs on Purpur only — keep those imports optional and brand-guarded.

## Official Resources (Always Check First)

### Paper
- **API Docs (stable)**: https://jd.papermc.io/paper/26.2/
- **API Docs (next, alpha)**: https://jd.papermc.io/paper/26.3/
- **Dev Docs**: https://docs.papermc.io/paper/dev/
- **Downloads API (v3)**: https://fill.papermc.io/v3/projects/paper
- **Maven Repository**: https://repo.papermc.io/
- **Downloads / News**: https://papermc.io/downloads · https://papermc.io/news/
- **Plugin Repository**: https://hangar.papermc.io/
- **paperweight-userdev (Gradle plugin)**: https://plugins.gradle.org/plugin/io.papermc.paperweight.userdev
- **Discord**: https://discord.gg/papermc

### Folia
- **Dev guide (Paper + Folia)**: https://docs.papermc.io/paper/dev/folia-support
- **Region overview / region logic**: https://docs.papermc.io/folia/reference/overview · https://docs.papermc.io/folia/reference/region-logic
- **Repository / README (broken API list)**: https://github.com/PaperMC/Folia
- **Downloads**: https://papermc.io/downloads/folia · https://fill.papermc.io/v3/projects/folia
- **API artifact**: `dev.folia:folia-api` from the PaperMC repo

### Purpur
- **Docs**: https://purpurmc.org/docs/purpur/ · **Configuration**: https://purpurmc.org/docs/purpur/configuration/
- **Permissions**: https://purpurmc.org/docs/purpur/permissions/ · **Commands**: https://purpurmc.org/docs/purpur/commands/
- **Javadoc**: https://purpurmc.org/javadoc/
- **Downloads API**: https://api.purpurmc.org/v2/purpur/ (v2, unlike Paper's sunset v2)
- **Maven**: https://repo.purpurmc.org/snapshots (`org.purpurmc.purpur:purpur-api`)
- **Repository / issues**: https://github.com/PurpurMC/Purpur · https://purpurmc.org/discord
- **Extras / Packs**: https://purpurmc.org/docs/purpurextras/ · https://purpurmc.org/docs/purpurpacks/

When you need a fact about a version, prefer the versioned Javadoc for that exact version and the Paper news post for that release — the "current" values in this file are a snapshot and new 26.x releases ship every few months.

## When to Read References

| Topic | Reference File |
|-------|---------------|
| Complete Maven/Gradle config, toolchain, test server | `references/project-setup.md` |
| plugin.yml / paper-plugin.yml full spec (incl. `folia-supported`) | `references/plugin-yml.md` |
| Events, commands, GUI, items, entities detailed examples | `references/api-patterns.md` |
| SQLite, MySQL, caching patterns | `references/data-storage.md` |
| Version matrix, api-version rules, migration, NMS | `references/version-matrix.md` |
| **Folia**: regionised threading, schedulers, thread ownership, broken API | `references/folia.md` |
| **Purpur**: fork features, `org.purpurmc.purpur` API, `purpur.yml`, permissions | `references/purpur.md` |
