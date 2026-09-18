# Version Compatibility & Migration Reference

## Table of Contents
1. [Version Matrix](#version-matrix)
2. [Paper Versioning Scheme (26.1+)](#versioning-scheme)
3. [Paper Hardfork (1.21.4+)](#hardfork)
4. [Mojang Mappings (1.20.5+)](#mojang-mappings)
5. [Java 25 Requirements](#java-25)
6. [NMS / Reflection](#nms-reflection)
7. [api-version Rules](#api-version)
8. [Folia Compatibility](#folia)
9. [Migration Guide](#migration-issues)
10. [26.x Breaking Changes & Deprecations](#breaking-changes)

---

## Version Matrix

| PaperMC Version | Minecraft Version | Min Java | api-version | Status | Key Changes |
|-----------------|-------------------|----------|-------------|--------|-------------|
| 1.20.5 – 1.20.6 | 1.20.5 – 1.20.6 | 21 | `1.20` | Old | Mojang-mapped runtime, CraftBukkit classes de-relocated |
| 1.21 – 1.21.11 | 1.21 – 1.21.11 | 21 | `1.21` | Old | Hardfork from Spigot begins at 1.21.4 |
| 26.1.1 | 26.1.1 | **25** | `26.1` | Unsupported (support ended 2026-04-11, last build 29 alpha) | Year-based versioning, new build/channel scheme |
| 26.1.2 | 26.1.2 | **25** | `26.1` or `26.1.2` | Supported (last build 74 stable, 2026-07-06) | World storage format change |
| **26.2** | **26.2** | **25** | **`26.2`** | **Latest stable** (first stable build #83, 2026-07-26; latest build #124, 2026-09-15) | Adventure 5, beds stop being block entities, `AbstractCubeMob` |
| 26.3 | 26.3 | 25 | `26.3` | **Paper alpha only** (builds 3–8 alpha; Minecraft 26.3 released 2026-09-15) | Poplars / dappled forest |

**Version numbering caveats:**

- Paper mirrors Minecraft version ids 1:1, but **not every id exists as a Paper version**: the 26.1 line contains only `26.1.1` and `26.1.2` — there is no Paper "26.1".
- Paper's *news post* date is not the stable-release date. The "26.2" news post is dated 2026-06-12 (same day as Minecraft 26.2-rc-2), while Minecraft 26.2 itself released 2026-06-16 and Paper's first stable 26.2 build shipped 2026-07-26.

**Minecraft version format change (2026):**

```
Old:  1.21.11      (1.x series)
New:  26.1 / 26.2 / 26.3   (year.drop[.patch])
```

Paper 26.2 = Minecraft Java 26.2. It is **not** 1.26.x and **not** 1.21.x. Never translate an old `1.x` number into `1.2x`.

Minecraft 26.3 ("Wilderness Bound") shipped 2026-09-15 (dappled forest biome, poplars, red shrubs, shelf mushrooms, wool/concrete stairs and slabs, straw beds, cushions, abandoned camps) and still targets **Java 25**. Paper 26.3 was still `-alpha` at the time of writing, so production plugins should target 26.2.

---

## Paper Family

Three servers matter in practice. They share Bukkit/Paper API but differ in threading model, API surface and build cadence.

| | **Paper** | **Folia** | **Purpur** |
|---|---|---|---|
| Relationship | Base | Paper fork (regionised multithreading) | Paper drop-in replacement (opt-in features) |
| Main thread | Yes | **No** — one tick loop per region, in parallel | Yes (not a Folia fork) |
| 26.2 status | `26.2.build.124-stable` | `26.2.build.7-beta` (**beta**) | `26.2.build.2633-stable` |
| 26.3 status | alpha | — | `26.3.build.2637-experimental` |
| Last stable line | 26.2 | 26.1.2 (build 8 stable) | 26.2 |
| Maven group | `io.papermc.paper` | `dev.folia` | `org.purpurmc.purpur` |
| Artifact | `paper-api` | `folia-api` | `purpur-api` |
| Repository | repo.papermc.io | repo.papermc.io | **repo.purpurmc.org/snapshots** |
| Pre-release channel name | `alpha` | `beta` | **`experimental`** |
| Downloads API | `fill.papermc.io/v3` (v2 sunset) | `fill.papermc.io/v3` | `api.purpurmc.org/v2` (still v2) |
| Extra opt-in for plugins | — | `folia-supported: true` (**required**) | none |
| Own API namespaces | `io.papermc.paper.*` | `io.papermc.paper.threadedregions.*` | `org.purpurmc.purpur.*` |

### Which target should a plugin choose?

| Plugin characteristic | Paper | Purpur | Folia |
|-----------------------|:----:|:------:|:-----:|
| Uses only Paper schedulers + `teleportAsync` | ✅ | ✅ | ✅ (with flag) |
| Uses `Bukkit.getScheduler()` / `BukkitRunnable` | ✅ | ✅ | ❌ |
| Uses scoreboard API | ✅ | ✅ | ❌ broken |
| Creates/unloads worlds at runtime | ✅ | ✅ | ❌ broken |
| Uses `Entity#teleport` (sync) | ✅ | ✅ | ❌ use `teleportAsync` |
| Imports `org.purpurmc.purpur.*` | ❌ | ✅ | ❌ |
| Declares `folia-supported: true` | ignored | ignored | required to load |

**Compile against Paper unless you truly need a fork API.** The four schedulers and `isOwnedByCurrentRegion` exist on Paper as well, so a Paper-only-compiled JAR is the most portable artifact. See [folia.md](folia.md) and [purpur.md](purpur.md).

### Build coordinates

```kotlin
// Paper (default)
compileOnly("io.papermc.paper:paper-api:26.2.build.124-stable")

// Folia
compileOnly("dev.folia:folia-api:26.2.build.7-beta")

// Purpur (includes Paper + Pufferfish + Spigot + Bukkit API)
repositories { maven("https://repo.purpurmc.org/snapshots") }
compileOnly("org.purpurmc.purpur:purpur-api:26.2.build.2633-stable")
```

### Version lookup endpoints

| Server | Latest stable version | Latest stable build |
|--------|----------------------|---------------------|
| Paper | `https://fill.papermc.io/v3/projects/paper` | `…/versions/26.2/builds` → parse for `channel == "STABLE"` |
| Folia | `https://fill.papermc.io/v3/projects/folia` | `…/versions/26.2/builds` (currently all `BETA`) |
| Purpur | `https://api.purpurmc.org/v2/purpur/` (`metadata.current`) | `https://api.purpurmc.org/v2/purpur/26.2` (`builds.latest`) |

---

## Versioning Scheme

Since 26.1, Paper artifact versions and build channels look like this:

```
<minecraft-version>.build.<build-number>-<channel>
       26.2        .build.   124       -stable
```

Channel semantics are officially defined by Paper (they replaced the older `experimental` / `default` naming):

| Channel | Official meaning | Guidance |
|---------|------------------|----------|
| `-alpha` | Used to be called "experimental" | Prone to error, receives no support — experimentation only |
| `-beta` | Middle point: not entirely unstable but partially unfinished (may lack config options or datafixer changes) | Early compatibility testing |
| `-stable` | Used to be called "default"; fixes and changes keep being pushed here | **The only channel to serve and use in production** ("You should always serve & use the stable builds") |

`recommended` exists in the downloads service but is **not used by Paper** (it is used by Velocity).

Worked examples from the `paper-api` Maven metadata (probed 2026-09-16):

| Version line | Alpha from | Beta from | Stable from | Latest build |
|--------------|-----------|-----------|-------------|--------------|
| 26.1.2 | build 2 | build 48 (2026-04-26) | build 53 (2026-05-01) | 74 stable (2026-07-06) |
| 26.2 | build 10 | build 59 (2026-07-12) | build 83 (2026-07-26) | 124 stable (2026-09-15) |
| 26.3 | build 1 | — | — | 8 alpha (2026-09-16) |

- `26.2.build.124-stable` ← current latest **stable**
- `26.1.2.build.74-stable` ← last stable of the 26.1 line
- `26.3-pre-2.build.0-alpha` ← newest published artifact overall, alpha only

Machine-readable sources:

- Maven metadata: `https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml`
- Downloads API **v3**: `https://fill.papermc.io/v3/projects/paper`, `…/versions/26.2/builds` (REST) or `https://fill.papermc.io/graphql`
- The old `https://api.papermc.io/v2/...` API is **sunset** (HTTP 410, `{"error":"sunset"}`) and must not be used in tooling. The v3 API asks for a descriptive `User-Agent` with contact info.

⚠️ In the Maven metadata, `<latest>` and `<release>` both point at `26.3-pre-2.build.0-alpha`. **`<release>` is not "the latest stable"** here — resolve the newest `-stable` entry by parsing the version list, don't trust those two tags.

**Pinning advice:** Gradle's documented form is `26.2.build.+` (the literal `build` token must stay — `26.2.+` could resolve to a different patch line). Maven's range form `[26.2.build,)` carries Paper's explicit **"Maven (Discouraged)"** label; prefer an exact `-stable` build id in `pom.xml` for reproducible builds.

---

## Hardfork

Since December 2024 (1.21.4), Paper is an independent project (hardfork from Spigot).

### What Changed

1. **Source structure**: API + implementation now live as real source files in the repo (not patches)
2. **Commit history**: Merged git trees with full patch-file history
3. **Version branches**: Old branches (`ver/1.8.8` to `ver/1.21.3`) moved to an archive repo
4. **Development speed**: Faster updates — can work on snapshots, pre-releases, release candidates

### What Did NOT Change

1. **Existing API**: All Spigot-inherited API methods still work
2. **Config files**: `bukkit.yml`, `spigot.yml`, `paper-global.yml` still work
3. **Plugin compatibility**: Plugins compiled against older Paper/Spigot still run (subject to `minimum-api`)
4. **Plugin loading**: `plugin.yml` format unchanged

### For Plugin Developers

- **Compile against Paper-API** (not the Spigot API) for 26.x
- Paper-only plugins are viable (Paper has ~85–90% market share)
- Publish on Hangar (https://hangar.papermc.io/) and Modrinth
- Spigot may add API after the hardfork that Paper does NOT automatically pull

---

## Mojang Mappings

Since Paper 1.20.5, the server jar/runtime uses Mojang's official mappings and CraftBukkit classes are no longer relocated into versioned packages.

### Impact on NMS Code

```java
// BEFORE (Spigot mappings, obfuscated + versioned package):
Class<?> nmsPlayer = Class.forName("net.minecraft.server.v1_21_R1.EntityHuman");

// AFTER (Mojang mappings, 1.20.5+ including all 26.x):
Class<?> nmsPlayer = Class.forName("net.minecraft.world.entity.player.Player");

// RECOMMENDED: use the paperweight-userdev Gradle plugin for NMS development
// It handles mappings at compile time, so no reflection is needed.
```

### Mappings Namespace in the Manifest

Paper decides how to treat your JAR from the `paperweight-mappings-namespace` manifest attribute in `META-INF/MANIFEST.MF`:

| Plugin type | Default assumption |
|-------------|--------------------|
| Bukkit/Spigot `plugin.yml` plugin | `spigot` (server deobfuscates the JAR on first load) |
| Paper `paper-plugin.yml` plugin | `mojang` (no remap needed) |

To make a `plugin.yml` plugin Mojang-mapped (recommended on 26.x — skips the one-time remap and keeps compatibility across minor updates at the cost of Spigot compatibility):

```kotlin
// build.gradle.kts
tasks.jar {
    manifest {
        attributes["paperweight-mappings-namespace"] = "mojang"
    }
}
// repeat for tasks.shadowJar if you use the shadow plugin
```

`paperweight-userdev` sets this attribute automatically, and you can also set it explicitly via `ReobfArtifactConfiguration.MOJANG_PRODUCTION`. Do not confuse this manifest attribute with `api-version`.

---

## Java 25

PaperMC 26.x requires **Java 25 minimum** — to run the server and to compile plugins.

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

### Build Configuration

```xml
<!-- Maven: use release, not source/target, so the JDK API level is enforced -->
<maven.compiler.release>25</maven.compiler.release>
```

```kotlin
// Gradle
java {
    toolchain.languageVersion.set(JavaLanguageVersion.of(25))
}
```

Paper itself builds with `JavaLanguageVersion.of(25)` **and** `options.release = 25`, so `paper-api` 26.x classes are class-file major **69**. Class-file majors for reference: 21 → 65, 22 → 66, 23 → 67, 24 → 68, **25 → 69**, 26 → 70. Compiling with a Java ≤ 21 toolchain therefore fails outright; that is why the version check error appears at build time as well as at server start.

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
        // Mojang mappings (1.20.5+, including 26.x)
        try {
            return Class.forName("net.minecraft." + className);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException("Cannot find class: " + className);
        }
    }
}
```

Reflection still needs raw names, which is exactly why the supported path is compiling against a dev bundle instead (see [paperweight-userdev](https://docs.papermc.io/paper/dev/userdev) and https://github.com/jpenilla/reflection-remapper if you must reflect).

### paperweight-userdev (Recommended for NMS)

Current version: **2.0.0-beta.23** (Gradle Plugin Portal, published 2026-08-28). The old `1.7.1` example seen in many tutorials is outdated.

```kotlin
plugins {
    id("io.papermc.paperweight.userdev") version "2.0.0-beta.23"
}

dependencies {
    // Dev bundle notation: paperweight.paperDevBundle(...) — different from the
    // Gradle string notation docs show generically as paperweight.paperDevBundle("26.2.build.+")
    // The dev bundle already contains the Paper API, so remove any separate
    // paper-api dependency when you use it.
    paperweight.paperDevBundle("26.2.build.124-stable")
}
```

Folia is supported through `paperweight.foliaDevBundle(...)`, and forks through `paperweight.devBundle("com.example.paperfork", "...")`.

Add the Paper repo to `settings.gradle.kts` if you need SNAPSHOT builds:

```kotlin
pluginManagement {
    repositories {
        gradlePluginPortal()
        maven("https://repo.papermc.io/repository/maven-public/")
    }
}
```

This provides compile-time access to Mojang-mapped NMS classes without reflection. Only the latest paperweight version is officially supported.

Paper's official example plugin currently pins `2.0.0-beta.21` with `paperDevBundle("26.1.2.build.+")`, `options.release = 25`, and the `xyz.jpenilla.run-paper` plugin for test servers — a good modern reference.

### Reobfuscation is gone from 26.1 onward

Mojang ships unobfuscated jars from 26.1, and Paper no longer reobfuscates to Spigot mappings, so the `reobfJar` task is reference-only for ≤ 1.21.11. On 26.x you ship **Mojang-mapped** plugins; there is no reobf step to configure. Do not confuse the `io.papermc.paperweight.userdev` plugin with `io.papermc.paperweight.core` (Paper's own build plugin).

---

## api-version

### Format Rules

Paper parses `api-version` in `org.bukkit.craftbukkit.util.ApiVersion`:

```java
// must be "major.minor" or "major.minor.patch", all parts numeric
String[] parts = versionString.split("\\.");
if (parts.length != 2 && parts.length != 3) {
    throw new IllegalArgumentException("API version string should be of format \"major.minor\"…");
}
```

So `26.2` is `major=26, minor=2, patch=0`, and `26.1.2` is a *different* version (`major=26, minor=1, patch=2`). They are not interchangeable.

**There is no hardcoded allow-list.** `ApiVersion` accepts any 2–3-part numeric string; the server only range-checks it against `CURRENT` (from Paper's own `gradle.properties` `apiVersion`, surfaced as `apiVersioning.json`) and the optional `settings.minimum-api` floor. That is why full-patch values such as `26.1.2` load fine, and why `26.3` is rejected on a 26.2 server.

Paper's own `gradle.properties` on the 26.2 branch reads:

```properties
mcVersion=26.2
apiVersion=26.2   # the current API version for use in (paper-)plugin.yml files
channel=STABLE
```

so **`26.2` is the canonical value** for a 26.2 plugin.

Validation performed by `CraftMagicNumbers#checkSupported(PluginDescriptionFile)`:

| Condition | Result |
|-----------|--------|
| `api-version` newer than the server's API version | `InvalidPluginException: Unsupported API version <value>` — plugin does not load |
| `api-version` older than `settings.minimum-api` (`bukkit.yml`, default `none`) | `InvalidPluginException: Plugin API version … is lower than the minimum allowed version` |
| `api-version` absent (`ApiVersion.NONE`) | loads as a legacy plugin with a warning; may enable Legacy Material Support |
| malformed value (e.g. `26.2.build.124-stable`, `1.26`, `latest`) | `IllegalArgumentException` while parsing |

Named `ApiVersion` constants are behaviour gates, not allowed values: `FLATTENING=1.13`, `FIELD_NAME_PARITY=1.20.5`, `ABSTRACT_COW=1.21.5`, `ABSTRACT_CUBE_MOB=26.2`.

### Values by Server Version

Paper's `plugin.yml` docs state the valid range is "1.13 – latest Paper release", and that minor versions have been supported since 1.20.5.

| Server Version | Recommended api-version |
|---------------|------------------------|
| 1.13 – 1.13.2 | `1.13` |
| 1.14 – 1.14.4 | `1.14` |
| 1.15 – 1.15.2 | `1.15` |
| 1.16 – 1.16.5 | `1.16` |
| 1.17 – 1.17.1 | `1.17` |
| 1.18 – 1.18.2 | `1.18` |
| 1.19 – 1.19.4 | `1.19` |
| 1.20 – 1.20.4 | `1.20` |
| 1.20.5 – 1.20.6 | `1.20.5` (minor versions supported since 1.20.5) |
| 1.21 – 1.21.11 | `1.21` |
| 26.1.1 / 26.1.2 | `26.1` (or the exact patch, e.g. `26.1.2`) |
| **26.2** | **`26.2`** |
| 26.3 | `26.3` (expected; Paper 26.3 is alpha) |

**Practical rule:** declare the oldest 26.x version that provides every API you use. `26.1` lets the plugin load on 26.1.2 and 26.2; `26.2` restricts it to 26.2+. Note there is no Paper "26.1" *build* — the api-version `26.1` simply means "needs at least the 26.1 API", which both 26.1.1 and 26.1.2 satisfy.

---

## Folia

> Full guide: [folia.md](folia.md) — regionised threading model, all four schedulers, thread-ownership checks, the broken-API list and a migration checklist. This section is only the summary.

Paper 26.x exposes Folia's regionised scheduler API on regular Paper too (`FallbackRegionScheduler`), so code written against `getRegionScheduler()` / `getGlobalRegionScheduler()` / `getAsyncScheduler()` / `Entity#getScheduler()` runs on both.

Folia-specific essentials:

- **Opt-in is mandatory**: without `folia-supported: true` in `plugin.yml`, Folia refuses to load the plugin.
- **No main thread**: `Bukkit.isPrimaryThread()` is not a meaningful test; use `Bukkit.isOwnedByCurrentRegion(...)` or `Bukkit.isGlobalTickThread()`.
- **`Entity#teleport` will never work** — use `Entity#teleportAsync`.
- **Scoreboard API, world load/unload, portals and respawn are broken.**
- Detect it with `ServerBuildInfo.buildInfo().isBrandCompatible(Key.key("papermc", "folia"))`.

### Folia-Safe Scheduling

```java
// Prefer the Paper schedulers: they work on Paper and Folia alike.
// - getGlobalRegionScheduler(): server-wide, main thread
// - getRegionScheduler():      tied to a Location/chunk's owning region
// - getAsyncScheduler():       off-thread work
public void giveCoins(Plugin plugin, Player player, int coins) {
    plugin.getServer().getGlobalRegionScheduler().run(plugin, task -> {
        player.giveExp(coins); // must touch the player on its owning thread
    });
}

public void updateBlockLater(Plugin plugin, Location location) {
    plugin.getServer().getRegionScheduler().runDelayed(plugin, location, task -> {
        location.getBlock().setType(Material.STONE);
    }, 20L);
}

// Detecting the real Folia server (regionised threading enabled):
private boolean isFolia() {
    try {
        Class.forName("io.papermc.paper.threadedregions.RegionizedServer");
        return true;
    } catch (ClassNotFoundException e) {
        return false;
    }
}
```

On plain Paper `Bukkit.getScheduler()` still works; it is simply not Folia-compatible.

---

## Migration Guide

### From 1.21.x to 26.1/26.2

1. **Change `api-version`** from `'1.21'` to `'26.1'` or `'26.2'`
2. **Change the Java target** from 21 to 25 (both compile and runtime)
3. **Update the Paper API dependency** to a 26.x `-stable` build (e.g. `26.2.build.124-stable`)
4. **Update libraries** — see the Adventure 5 section below; text/item APIs moved
5. **Back up worlds** — world storage format changed in 26.1 and **cannot be downgraded** after upgrade
6. **Test thoroughly** — the hardfork and the version bump both carry behavior changes

### From 26.1.x to 26.2.x

1. **Leave `api-version: '26.1'`** if you do not use new 26.2 API — this keeps 26.1 compatibility
2. **Migrate Adventure 4 → 5** (see below)
3. **Stop storing PDC data on beds** — beds are no longer block entities
4. **Rebuild against `26.2.build.XXX-stable`** and re-test event handlers for cube mobs and dripstone blocks
5. Do not downgrade a 26.2 world back to 26.1

### Legacy Material Support Warning

If you see:
```
[STDERR] CraftLegacy Initializing Legacy Material Support.
```

Causes:
- Missing `api-version` in `plugin.yml`
- An `api-version` older than 1.13 (`ApiVersion.FLATTENING`)
- Plugin compiled against a very old API

Fix: set a correct modern `api-version` (e.g. `'26.2'`).

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

- `PlayerChatEvent` → use `AsyncChatEvent`
- The whole `org.bukkit.conversations` API → deprecated for removal; use `AsyncChatEvent` or `Dialog` for input
- `ChatColor` / `setDisplayName(String)` / `setLore(List<String>)` → Adventure `Component` / MiniMessage
- `co.aikar.timings.*` → terminally deprecated; profile with **spark**
- `Bukkit.getLogger()` → use `plugin.getLogger()` for prefix identification

---

## Breaking Changes & Deprecations

Confirmed against the Paper 26.2 and 26.3 Javadocs and Paper's release notes.

### 26.2 (latest stable)

| Change | Impact | Action |
|--------|--------|--------|
| **Beds are no longer block entities** | `org.bukkit.block.Bed` is `@Deprecated(forRemoval=true, since="26.2")` — "bed block entity no longer exists". Beds cannot hold a `PersistentDataContainer`; `Bed#setColor(DyeColor)` now throws `UnsupportedOperationException` ("not supported, set the block type"). | Move bed data elsewhere; listen to `AsyncServerDataFixerRemoveBlockEntityEvent`. Do not confuse `org.bukkit.block.Bed` (deprecated `TileState`) with the still-supported `org.bukkit.block.data.type.Bed` BlockData interface. |
| **Adventure 5** | Removed API that was deprecated in Adventure 4 | `BookMeta` no longer extends Adventure's `Book` and `BookMeta#toBuilder()` is gone — use `BookMeta`'s own `pages(...)`, `title(...)`, `author(...)`, `page(int, Component)` and `asBook()`; stop using removed `ClickEvent`/`HoverEvent` forms. See [Adventure 5 details](#adventure-5). |
| `PointedDripstone` block data | `@Deprecated(forRemoval=true, since="26.2")` — "multiple type of speleothem exists now" | Use `Speleothem` (`getVerticalDirection()`, `getThickness()`, `Speleothem.Thickness`); `PointedDripstone` just extends it and declares nothing. There is no `Dripstone` type. |
| `PinkPetals` block data | Deprecated ("multiple flower collection blocks exist now") | Use `FlowerBed` |
| `MagmaCube extends Slime` | **Hard break, source and binary**: `MagmaCube` now extends `AbstractCubeMob, Enemy` | Handle `AbstractCubeMob`; `SlimeSplitEvent#getEntity()` returns `AbstractCubeMob` (erased descriptor changed → `NoSuchMethodError` for plugins compiled on 26.1) |
| `Vex#getSummoner()` / `setSummoner(Mob)` | Deprecated for removal in 26.2 | Use `Vex#getOwner()` / `setOwner(LivingEntity)` |
| `World#setSpawnFlags(...)`, `World#getAllowAnimals()` | Deprecated for removal — "the vanilla server no longer maintains this functionality" | Drop the calls |
| `CreeperIgniteEvent` | Now extends the new `EntityIgniteEvent` (package `io.papermc.paper.event.entity`), which also fires for sulfur cubes and adds `getFuseTime()` / `setFuseTime(int)` | Listen to `EntityIgniteEvent` for generic ignite logic |
| New registries | `RegistryEvents.TRIM_MATERIAL` / `TRIM_PATTERN` | Use them instead of hardcoding trim data |
| Client brand | `PlayerCommonConnection#getClientBrandName()` (nullable; `vanilla` for the Notchian client) | Available for connection-level logic |
| `CreakingHeart#isActive/setActive` | Deprecated | Use `getCreakingHeartState()` / `setCreakingHeartState(State)` |
| `Vault#getTrialSpawnerState/setTrialSpawnerState` | Deprecated | Use `getVaultState()` / `setVaultState(State)` |
| `BlockSoundGroup` (destroystokyo) | Deprecated | Use `org.bukkit.SoundGroup` |
| `TargetBlockInfo`, `BlockSoundGroup` ray-trace helpers | Deprecated | Use `RayTraceResult`, `FluidCollisionMode` |
| `BukkitBrigadierCommand` / `PaperBrigadier` | Deprecated for removal | Use `io.papermc.paper.command.brigadier.Commands` and `LifecycleEvents.COMMANDS` |
| `EntityState` teleport flags (`RETAIN_PASSENGERS`, `RETAIN_OPEN_INVENTORY`, `RETAIN_VEHICLE`) | Behavior is now vanilla default / removed | Stop passing them; close inventories manually |
| `Mob#setDespawnInPeacefulOverride` | Deprecated ("clients no longer render those entities") | Avoid |
| Concrete custom tags | Deprecated in favour of vanilla tags | Use the vanilla tag constants |
| `io.papermc.paper.text.PaperComponents.*` serializers | Terminally deprecated | Use `GsonComponentSerializer.gson()`, `LegacyComponentSerializer.legacySection()`, `PlainTextComponentSerializer.plainText()`, `GsonComponentSerializer.colorDownsamplingGson()` |
| Behaviour: data-component/spawn edge cases | Runtime fixes during the 26.2 cycle: `CraftEntity` data-component conversion threw `ClassCastException`; `ItemContainerContents#contents` could return null items instead of empty ones; `RegionAccessor#spawn` threw `IllegalArgumentException` for slime-family entities | Use a 26.2 build ≥ #67 if you rely on these |
| Behaviour: `InventoryClickEvent` | Now only pre-cancelled for spectators | Don't assume spectator clicks arrive pre-cancelled for other cases |

#### Adventure 5

Paper 26.2 ships Adventure **5.x** (the 26.2 Javadocs link to Adventure 5.2.0). Most impactful removals for plugin authors:

- `BookMeta` no longer extends Adventure's `Book`, and `BookMeta#toBuilder()` / `BookMeta.BookMetaBuilder` are **gone**. `BookMeta` is mutable — set pages directly:

  ```java
  // 26.1.x (builder — removed in 26.2)
  BookMeta built = meta.toBuilder().title(t).pages(p).build();

  // 26.2+
  BookMeta meta = (BookMeta) item.getItemMeta();
  meta.title(Component.text("Guide"));
  meta.author(Component.text("Me"));
  meta.pages(List.of(Component.text("page 1"), Component.text("page 2")));
  item.setItemMeta(meta);

  // Need an Adventure Book (e.g. Audience#openBook)?
  net.kyori.adventure.inventory.Book book = meta.asBook();
  ```

  This is a real-world break: plugins that called the inherited `Book` setters compile on 26.1.x and die with `NoSuchMethodError` on 26.2 ([Paper issue #14030](https://github.com/PaperMC/Paper/issues/14030)).

- `ClickEvent` is a typed interface; `ClickEvent.Action` is no longer an enum. `ClickEvent#value` and `ClickEvent#create(Action, String)` are removed — use `payload()` or the direct factories (`ClickEvent.openUrl(String)`).
- `BuildableComponent` removed → `Component#toBuilder()`; `NBTComponent` now takes one type argument.
- `Audience#sendMessage` overloads taking `Identity`/`Identified` removed; the `MessageType` enum removed entirely.
- Boss-bar percent API removed → progress constants/methods.
- Non-builder `Component#join` and `replaceText` overloads removed → pass `JoinConfiguration` / `TextReplacementConfig`.
- `TranslationRegistry` → `TranslationStore`; `PlainComponentSerializer` → `PlainTextComponentSerializer`; `JSONComponentConstants` → `ComponentTreeConstants`.
- `GSONComponentSerializer` options `downsampleColors` / `emitLegacyHoverEvent` removed → `JSONOptions.EMIT_RGB` / `JSONOptions.EMIT_HOVER_EVENT_TYPE`.
- `TextFormat` is sealed; custom `Component` implementations are impossible (use `VirtualComponent`); `ChatType` no longer implements `Keyed` and `ChatType#key` is nullable.
- `adventure-extra-kotlin` and `adventure-text-serializer-gson-legacy-impl` modules removed; Adventure requires Java 21+.

**MiniMessage has no 26.2-specific break** — MiniMessage usage is unaffected by the Adventure 5 update.

### 26.1

- **World storage format changed** — upgrading a world to 26.1+ is irreversible; Paper explicitly warns you cannot downgrade afterwards. Back up before upgrading:

  ```
  world/
  ├── data/minecraft/…
  ├── datapacks/
  ├── dimensions/minecraft/{overworld,the_nether,the_end}/
  │   ├── data/minecraft/…            # weather.dat, world_gen_settings.dat, …
  │   ├── data/paper/                 # level_overrides.dat, metadata.dat,
  │   │                               # persistent_data_container.dat
  │   ├── entities/  poi/  region/
  │   └── paper-world.yml             # moved out of the server root
  ├── players/{advancements,data,stats}/
  └── level.dat
  ```

  Dimensions moved from separate folders in the server root into `world/dimensions/`, and per-world `paper-world.yml` files moved into the dimension folders. If your plugin reads or writes world files by path, this breaks — world-level PDC is now `world/dimensions/minecraft/<dim>/data/paper/persistent_data_container.dat`, and hardcoded paths such as `world/persistent_data_container.dat`, `world/paper-world.yml`, `world_nether/`, `world_the_end/` or `world/DIM-1` must be updated.

- **World keys replace world names** — `WorldInfo#getName()` and related API are marked obsolete and may be deprecated. Use the `getKey()` methods instead, because Paper is moving away from world *names* internally.

- **Per-world time / `ClockTimeSkipEvent`** — clocks are registry-driven (not modifiable after startup), so Paper makes all clocks **per-world** by default; toggle with the new `time.affects-all-worlds` config option. `org.bukkit.event.world.ClockTimeSkipEvent` (with a nested `ClockTimeSkipEvent.SkipReason` enum) is the new parent of `TimeSkipEvent`. ⚠️ When `time.affects-all-worlds` is enabled (vanilla behavior), **only the new parent event is called for most command and API actions** — plugins listening only to `TimeSkipEvent` will silently stop firing.

- **Unobfuscated server jars / remapper removed** — Mojang stopped shipping obfuscated server jars, so Paper **fully dropped the internal remapper**; obfuscated names no longer exist in the server jar. From 26.1 on, Paper does not support obfuscated plugins and `reobfJar` no longer works for 26.1+ dev bundles. Ship Mojang-mapped.

- **Versioning change** — `-R0.1-SNAPSHOT` replaced by the build/channel scheme described above.
- Expect a harmless `Missing file: /data/minecraft/game_rules.dat` log line on 26.1/26.2 servers (single-player-only logic).

#### New 26.1 API worth knowing

`World#locateNearestPoi` / `locateAllPoiInRange`, `WorldCreator#forcedSpawnPosition`, `PositionedRayTraceConfigurationBuilder#blockCollisionMode`, `EntityLungeEvent`, `PlayerToggleEntityAgeLockEvent`, `PlayerSwapWithEquipmentSlotEvent`, `ItemCraftedEvent`, `PlayerPurchaseEvent#getMerchant`, `FurnaceExtractEvent#getItemStack`, `VehicleDamageEvent#getDamageSource` / `VehicleDestroyEvent#getDamageSource`, `RecipeChoice.ItemTypeChoice`, `Mannequin.validPoses`, `Raid#setTotalWaves`, `GameRule#getDefaultValue`, `Damageable#kill`.

New server options: `add-plugin-dir` startup argument, `misc.max-tracking-combat-entries`, `entities.spawning.max-arrow-despawn-invulnerability`, `spam-limiter.incoming-packet-treshold` (`-1` disables), `unsupported-settings.ticking.chunks` / `.blockEntities`, and `region-file-compression` in `server.properties` (replacing Paper's `unsupported-settings.compression-format`, now also allowing `gzip`).

### 26.3 (alpha)

Still changing daily — **do not ship against it**. Known so far:

- `TreeType.POPLAR` added (Minecraft 26.3 adds poplars and the dappled forest).
- New package `io.papermc.paper.block.pot` with `PotPatternType` / `PotPatternTypes` (decorated-pot patterns).
- **Particle API rework**: `ParticleBuilder`'s `extra` getter/setter deprecated, `spawnParticle` expanded, new randomization enum.
- Registry **holder-set rework** is an unmerged draft (`RegistryValueSet` → `RegistryHolderSet`, new `RegistryFactory`) — not shipped API, do not code against it.
- Minecraft 26.3 adds **straw beds**, which raises the same "is it a block entity?" question as regular beds in 26.2.
- No official Paper 26.3 announcement exists yet; there is no `26.3` git tag and no stable build.

### Community-reported breakages on 26.x

Real reports worth checking your own plugin for:

| Symptom | Cause | Fix |
|---------|-------|-----|
| `NoSuchMethodError` on `BookMeta` page/title setters after upgrading | Adventure 5 moved those methods from Adventure's `Book` to `BookMeta` ([#14030](https://github.com/PaperMC/Paper/issues/14030)) | Call `BookMeta`'s own methods / `asBook()` |
| `IllegalArgumentException: Cannot spawn an entity for org.bukkit.entity.AbstractCubeMob` | `MagmaCube` rewrite to `AbstractCubeMob` broke `spawn` ([#13978](https://github.com/PaperMC/Paper/issues/13978), fixed in build #30) | Spawn `Slime`/`MagmaCube` types, and use a 26.2 build with the fix |
| `ArrayIndexOutOfBoundsException: Index 3 out of bounds for length 3`, or "unknown/unsupported server version" | Plugin parses the old `v1_20_R3`-style CraftBukkit package name (relocation dropped in 1.20.5) | Use `Bukkit.getServer().getMinecraftVersion()` / `getBukkitVersion()`, or `getKey()`-based APIs |
| `UnsupportedClassVersionError`, or "requires running the server with Java 25 or above", `[ERROR] Could not load plugin` | Server or plugin built for Java < 25 | Update the server JRE and the plugin's toolchain/release to 25 |
| `InvalidPluginException: Unsupported API version …` | `api-version` newer than the server (e.g. `26.3` on a 26.2 server) | Lower `api-version` to what the server actually provides |
| "Plugin Not Compatible With This Version" | `api-version` below `settings.minimum-api` | Raise it, or relax the server's `minimum-api` |
| `TimeSkipEvent` handlers stopped firing | `ClockTimeSkipEvent` is now the parent, and with `time.affects-all-worlds` only the parent fires | Listen to `ClockTimeSkipEvent` |
| Plugin suddenly throws on previously tolerated input | 26.2 added argument validation (`add preconditions to particles`, `clean block placement from API`) | Validate arguments yourself |

### Long-standing terminal deprecations still present in 26.x

- `co.aikar.timings.*` (whole package) — removal planned, use spark.
- `org.bukkit.conversations.*` (whole package) — use `AsyncChatEvent` / `Dialog`.
- Keyed "enum-like" types (`Biome`, `Art`, `Attribute`, `PatternType`, `Cat.Type`, `Frog.Variant`, …): `valueOf`/`values` retained for compatibility only — use `Registry.get(NamespacedKey)` / `Registry.stream()`.
- `AttributeModifier` UUID constructors and `AttributeInstance#getModifier(UUID)`/`removeModifier(UUID)` — attributes are keyed now; use `net.kyori.adventure.key.Key` overloads.
