# Version Compatibility & Migration Reference

## Table of Contents
1. [Version Matrix](#version-matrix)
2. [Paper Hardfork (1.21.4+)](#hardfork)
3. [Mojang Mappings (26.1+)](#mojang-mappings)
4. [Java 25 Requirements](#java-25)
5. [NMS / Reflection](#nms-reflection)
6. [api-version Values](#api-version)
7. [Folia Compatibility](#folia)
8. [Common Migration Issues](#migration-issues)

---

## Version Matrix

| PaperMC Version | Minecraft Version | Min Java | api-version | Key Changes |
|-----------------|-------------------|----------|-------------|-------------|
| 1.20.5 - 1.20.6 | 1.20.5 - 1.20.6 | 21 | `1.20` | Mojang-mapped server jar |
| 1.21 - 1.21.3 | 1.21 - 1.21.3 | 21 | `1.21` | Hardfork begins |
| **26.1.x** | **26.1.x** | **25** | **`26.1.2`** | **Hardfork complete, year-based versioning** |
| 26.2.x | 26.2.x | 25 | `26.2` | Future release |

**Minecraft Version Format Change (2026):**
- Old: `1.21.11` (1.x series)
- New: `26.1.2` (year.drop.patch)

PaperMC 26.1.2 = Minecraft Java 26.1.2. This is NOT 1.21.x.

---

## Hardfork

Since December 2024 (1.21.4), Paper is an independent project (hardfork from Spigot).

### What Changed

1. **Source structure**: API + implementation now live as real source files in the repo (not patches)
2. **Commit history**: Merged git trees with full patch-file history
3. **Version branches**: Old branches (`ver/1.8.8` to `ver/1.21.3`) moved to archive repo
4. **Development speed**: Faster updates — can work on snapshots, pre-releases, release candidates

### What Did NOT Change

1. **Existing API**: All Spigot-inherited API methods still work
2. **Config files**: `bukkit.yml`, `spigot.yml`, `paper-global.yml` still work
3. **Plugin compatibility**: Plugins compiled against older Paper/Spigot still run
4. **Plugin loading**: `plugin.yml` format unchanged

### For Plugin Developers

- **Compile against Paper-API** (not Spigot API) for 26.1.2+
- Paper-only plugins are viable (Paper has ~85-90% market share)
- Publish on Hangar (https://hangar.papermc.io/) and Modrinth
- Spigot may add new API after hardfork that Paper does NOT automatically pull

---

## Mojang Mappings

Since Paper 1.20.5, the server jar uses Mojang's official mappings (no runtime remapping).

### Impact on NMS Code

```java
// BEFORE (Spigot mappings, obfuscated):
Class<?> nmsPlayer = Class.forName("net.minecraft.server.v1_21_R1.EntityHuman");

// AFTER (Mojang mappings):
Class<?> nmsPlayer = Class.forName("net.minecraft.world.entity.player.Player");

// RECOMMENDED: Use paperweight-userdev Gradle plugin for NMS development
// This handles mappings automatically at compile time
```

### Disable Plugin Remapping (Future)

Paper will eventually disable automatic plugin remapping at runtime. To prepare:

1. Use `paperweight-userdev` if you access NMS internals
2. Update reflection code to use Mojang class names
3. Test with `-Dpaper.disablePluginRemapping=true` flag

---

## Java 25

PaperMC 26.1.x requires **Java 25 minimum**.

### Check Java Version at Runtime

```java
@Override
public void onEnable() {
    int javaVersion = Runtime.version().feature();
    if (javaVersion < 25) {
        getLogger().severe("This plugin requires Java 25! Current: " + javaVersion);
        getServer().getPluginManager().disablePlugin(this);
        return;
    }
}
```

### Java 25 Features Useful for Plugins

```java
// Records for data models
public record PlayerData(UUID uuid, String name, int coins) {}

// Pattern matching for switch
String category = switch (material) {
    case DIAMOND, EMERALD -> "valuable";
    case STONE, DIRT -> "common";
    case null -> "null";
    default -> "other";
};

// Text blocks (multi-line strings)
String sql = """
    SELECT uuid, name, coins
    FROM players
    WHERE coins > ?
    """;

// Sealed classes
public sealed abstract class Transaction
    permits BuyTransaction, SellTransaction {}

// Virtual threads (caution: must switch to main thread for Bukkit API)
Thread.startVirtualThread(() -> {
    var data = database.load(uuid);
    Bukkit.getScheduler().runTask(plugin, () -> player.sendMessage("Done"));
});
```

---

## NMS / Reflection

### Dynamic Version Detection

```java
public class NMSUtil {
    public static Class<?> getNMSClass(String className) {
        // Try Mojang mappings (26.1+)
        try {
            return Class.forName("net.minecraft." + className);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("Cannot find class: " + className);
        }
    }
}
```

### Paperweight Userdev (Recommended for NMS)

For Gradle (requires accessing Mojang-mapped NMS at compile time):

```kotlin
plugins {
    id("io.papermc.paperweight.userdev") version "1.7.1"
}

dependencies {
    paperDevBundle("26.1.2.build.72-stable")
}
```

This provides compile-time access to Mojang-mapped NMS classes without reflection.

---

## api-version

### Values by Server Version

| Server Version | api-version Value |
|---------------|-------------------|
| 1.13 - 1.13.2 | `1.13` |
| 1.14 - 1.14.4 | `1.14` |
| 1.15 - 1.15.2 | `1.15` |
| 1.16 - 1.16.5 | `1.16` |
| 1.17 - 1.17.1 | `1.17` |
| 1.18 - 1.18.2 | `1.18` |
| 1.19 - 1.19.4 | `1.19` |
| 1.20 - 1.20.4 | `1.20` |
| 1.20.5 - 1.20.6 | `1.20` |
| 1.21 - 1.21.3 | `1.21` |
| **26.1.x** | **`26.1.2`** |

**Critical**: For PaperMC 26.1.2, use `api-version: '26.1.2'` in `plugin.yml`. This is confirmed by the official PaperMC documentation and production plugins. Using `'1.21'` is legacy and may cause issues.

---

## Folia

PaperMC 26.1 may use Folia's regionized threading model internally.

### Folia-Safe Scheduler

```java
// Check if Folia scheduler is available
try {
    Bukkit.getServer().getRegionScheduler();
    // Running on Folia-aware Paper
} catch (NoSuchMethodError e) {
    // Standard Paper — use Bukkit scheduler
}

// Folia-compatible pattern
public void runTask(Plugin plugin, Location location, Runnable task) {
    if (isFolia()) {
        Bukkit.getServer().getRegionScheduler().run(plugin, location, scheduledTask -> task.run());
    } else {
        Bukkit.getScheduler().runTask(plugin, task);
    }
}

private boolean isFolia() {
    try {
        Class.forName("io.papermc.paper.threadedregions.RegionizedServer");
        return true;
    } catch (ClassNotFoundException e) {
        return false;
    }
}
```

---

## Migration Issues

### From 1.21.x to 26.1.x

1. **Change `api-version`** from `'1.21'` to `'26.1.2'`
2. **Change Java target** from 21 to 25
3. **Update Paper API dependency** to `26.1.2.build.XX-stable`
4. **Test thoroughly** — hardfork may have subtle behavior changes

### Legacy Material Support Warning

If you see:
```
[STDERR] CraftLegacy Initializing Legacy Material Support.
```

Causes:
- Missing `api-version` in `plugin.yml`
- Wrong `api-version` value
- Plugin compiled against very old API

Fix: Set correct `api-version: '26.1.2'`.

### Material Name Changes

```java
// 1.20 -> 1.21+
Material.GRASS;        // May not exist
Material.SHORT_GRASS;  // Use this instead

// Always handle missing materials gracefully
try {
    Material mat = Material.valueOf(configString);
} catch (IllegalArgumentException e) {
    getLogger().warning("Unknown material: " + configString);
}
```

### Common Deprecations

- `PlayerChatEvent` → Use `AsyncChatEvent` (Paper's async chat event)
- Legacy color codes (`§a`, `ChatColor.GREEN`) → Consider Adventure Component API / MiniMessage
- `Bukkit.getLogger()` → Use `plugin.getLogger()` for prefix identification
