---
name: papermc-dev
description: PaperMC 26.1.2 (Minecraft Java 26.1.2) plugin development guide. Use when the user needs to develop, code, debug, or maintain PaperMC server plugins. Covers Java 25, Maven/Gradle project setup, plugin.yml/paper-plugin.yml configuration, Bukkit/Paper API (events, commands, schedulers, GUI, items, entities, worlds), data persistence (SQLite/MySQL), performance optimization, security best practices, and version compatibility. Also applies when user asks about Bukkit/Spigot/Paper plugin development, Minecraft server plugins, or migrating plugins to newer Paper versions.
---

# PaperMC Plugin Development Skill

Comprehensive guide for developing PaperMC 26.1.2 plugins using Java 25 and Maven/Gradle.

## Version Facts

- **PaperMC 26.1.2** corresponds to **Minecraft Java 26.1.2** (NOT 1.21.x). Mojang switched to year-based versioning in 2026.
- **Minimum Java**: 25. Using Java 21 or lower causes `UnsupportedClassVersionError`.
- **API dependency**: `io.papermc.paper:paper-api:26.1.2.build.+` (Maven scope: `provided`).
- **Hardfork**: Since 1.21.4 (2024-12), Paper is independent from Spigot. Existing Spigot API methods still work, but new plugins should compile against Paper-API.
- **api-version in plugin.yml**: Use `'26.1.2'` (confirmed by official docs and production plugins). Using `'1.21'` is legacy and may trigger Legacy Material Support.
- **Deobfuscated server jar**: Paper 26.1 uses Mojang mappings directly. NMS reflection must use Mojang class names (e.g., `net.minecraft.world.entity.player.Player`).

## Quick Start Workflow

### 1. Project Setup

Reference: [references/project-setup.md](references/project-setup.md) for complete Maven `pom.xml` and Gradle `build.gradle.kts` templates.

**Maven essentials:**
```xml
<properties>
    <maven.compiler.source>25</maven.compiler.source>
    <maven.compiler.target>25</maven.compiler.target>
    <paper.api.version>26.1.2.build.72-stable</paper.api.version>
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
api-version: '26.1.2'
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
- `api-version`: MUST be set to `'26.1.2'`. Omitting it triggers Legacy Material Support (slow startup, compatibility issues).
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
        Bukkit.getScheduler().cancelTasks(this);
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
6. Register commands + tab completers
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
        player.sendMessage("Welcome!");
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

### Scheduler (Critical Thread Safety)

| Operation | Required Thread |
|-----------|----------------|
| Modify blocks | Main (sync) |
| Operate entities | Main (sync) |
| Player inventory | Main (sync) |
| Database queries | Async |
| File I/O | Async |
| HTTP requests | Async |

```java
// Async task, then back to main thread
Bukkit.getScheduler().runTaskAsynchronously(plugin, () -> {
    PlayerData data = database.load(uuid);
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage("Loaded: " + data.getCoins());
    });
});
```

### GUI / Inventory

Create `InventoryHolder` implementation, open with `player.openInventory(inv)`. Handle `InventoryClickEvent` and `InventoryDragEvent`, always `event.setCancelled(true)` for GUI clicks. Use `PersistentDataContainer` to tag GUI items with action identifiers.

### ItemStack with Custom Data

```java
ItemStack item = new ItemStack(Material.DIAMOND_SWORD);
ItemMeta meta = item.getItemMeta();
meta.setDisplayName("Legendary Sword");
meta.setLore(Arrays.asList("Line 1", "Line 2"));
meta.getPersistentDataContainer().set(
    new NamespacedKey(plugin, "item_id"), PersistentDataType.STRING, "legendary_sword"
);
item.setItemMeta(meta);
```

## Data Persistence

Reference: [references/data-storage.md](references/data-storage.md) for complete SQLite/MySQL/Caffeine patterns.

**Rule of thumb:**
- Small server / simple data: SQLite (built-in JDBC, no extra dependency)
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

### Security
- [ ] All SQL via `PreparedStatement`
- [ ] Validate all command arguments (type, range, regex)
- [ ] File operations restricted to plugin data folder (check canonical path)
- [ ] Audit log for sensitive operations
- [ ] Permission check on every admin operation
- [ ] Avoid Java native serialization
- [ ] Whitelist reflection target classes
- [ ] Never expose credentials in logs

## Version Compatibility

Reference: [references/version-matrix.md](references/version-matrix.md) for full compatibility details.

| PaperMC | Minecraft | Min Java | api-version | Notes |
|---------|-----------|----------|-------------|-------|
| 1.20.5+ | 1.20.5+ | 21 | `1.20` | — |
| 1.21.x | 1.21.x | 21 | `1.21` | — |
| **26.1.x** | **26.1.x** | **25** | **`26.1.2`** | Hardfork, Mojang mappings |

**Paper vs Spigot after hardfork:**
- Existing Spigot API methods continue to work
- Paper may add new API not in Spigot
- Plugins compiled against Paper-API may not run on Spigot
- Paper market share is ~85-90%, making Paper-only plugins viable

## Official Resources (Always Check First)

- **API Docs**: https://jd.papermc.io/paper/26.1.2/
- **Dev Docs**: https://docs.papermc.io/paper/dev/
- **Repository**: https://repo.papermc.io/
- **Downloads**: https://papermc.io/downloads
- **Plugin Repository**: https://hangar.papermc.io/
- **Discord**: https://discord.gg/papermc

## When to Read References

| Topic | Reference File |
|-------|---------------|
| Complete Maven/Gradle config | `references/project-setup.md` |
| plugin.yml / paper-plugin.yml full spec | `references/plugin-yml.md` |
| Events, commands, GUI, items, entities detailed examples | `references/api-patterns.md` |
| SQLite, MySQL, caching patterns | `references/data-storage.md` |
| Version compatibility, migration, NMS | `references/version-matrix.md` |
