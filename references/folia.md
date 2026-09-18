# Folia Reference

**Folia** is PaperMC's fork of Paper that adds **regionised multithreading** to the dedicated server. It is a separate project and will not be merged into Paper "for the foreseeable future".

- Repository: https://github.com/PaperMC/Folia (default branch `ver/26.2.x`)
- Docs: https://docs.papermc.io/folia/reference/overview · https://docs.papermc.io/paper/dev/folia-support
- Downloads: https://papermc.io/downloads/folia · `https://fill.papermc.io/v3/projects/folia`
- Maven group: `dev.folia` (note: **not** `io.papermc.paper`)

## Status & Versions (2026-09-17)

| Folia line | Channel | Notes |
|-----------|---------|-------|
| 26.1.2 | build 8 **stable** | Last stable line |
| **26.2** | builds 1–7 **beta** | Latest, but still **BETA** — `26.2.build.7-beta` |

Folia lags Paper and is usually released later and with fewer builds. Verify the channel before recommending it for production:

```bash
curl https://fill.papermc.io/v3/projects/folia/versions/26.2/builds
```

## How Regionised Multithreading Works

Folia groups nearby loaded chunks into independent **regions**. Each region has its **own tick loop**, ticked at the normal 20 TPS, and the region tick loops run **in parallel on a thread pool**.

**There is no main thread.** Each region effectively has its own "main thread" that executes its entire tick loop. A single-threaded Paper server can be thought of as one giant region covering every chunk of every world.

Regionizer invariants that matter to plugin authors:

1. A ticking region **cannot expand** the chunks it owns while ticking.
2. A ticking region owns a buffer of chunks around its perimeter.
3. A region will not begin ticking if it has an adjacent ticking neighbour.
4. Adjacent regions eventually merge; large regions split when possible.

Consequences:

- `Bukkit.isPrimaryThread()` is effectively meaningless on Folia — there is no primary thread.
- Region-local counters: `Current Tick` and `Redstone Time` are **per region**; global game time and daylight time come from the **global region** and are copied at the start of each region tick.
- The **global region** is a single always-20-TPS task that owns things not tied to any region: game rules, global game time, daylight time, weather, world border, console command handling.

## Marking a Plugin as Folia-Supported

Folia **refuses to load** plugins that do not opt in. Add this to `plugin.yml` or `paper-plugin.yml`:

```yaml
folia-supported: true
```

> ⚠️ Setting this flag is *not nearly enough* to support Folia. Folia's own README puts compatibility expectations at **0** for unmodified Paper plugins, and adds "I expect basically zero plugins that are compatible with Paper to be compatible with Folia." Only set the flag once you have actually made the plugin region-safe.

## Detecting Folia

```java
import io.papermc.paper.ServerBuildInfo;
import net.kyori.adventure.key.Key;

private static boolean isFolia() {
    return ServerBuildInfo.buildInfo().isBrandCompatible(Key.key("papermc", "folia"));
}
```

For backwards compatibility, the older class-probe still works:

```java
private static boolean isFolia() {
    try {
        Class.forName("io.papermc.paper.threadedregions.RegionizedServer");
        return true;
    } catch (ClassNotFoundException e) {
        return false;
    }
}
```

## Schedulers

`BukkitScheduler` inherently depends on one main thread. Folia adds four schedulers that replace it; on regular Paper the same API is handled internally to behave like a single main thread, so **the same code works on both**.

| Scheduler | Access | Use for |
|-----------|--------|---------|
| `GlobalRegionScheduler` | `server.getGlobalRegionScheduler()` | Server-wide state: game rules, weather, console commands, global counters, world border |
| `RegionScheduler` | `server.getRegionScheduler()` | Work owned by the region that owns a **Location/chunk** |
| `AsyncScheduler` | `server.getAsyncScheduler()` | Off-thread work: DB, file I/O, HTTP |
| `EntityScheduler` | `entity.getScheduler()` | Work on a **specific entity** — follows it across regions |

```java
// Region-owned block change
Location locationToChange = ...;
server.getRegionScheduler().execute(plugin, locationToChange, () -> {
    locationToChange.getBlock().setType(Material.BEEHIVE);
});

// Entity work that follows the entity across regions
entity.getScheduler().run(plugin, scheduledTask -> {
    entity.setCustomName("Tracked");
}, null);

// Global (server-wide) work
server.getGlobalRegionScheduler().execute(plugin, () -> {
    server.getWorld("world").setStorm(false);
});

// Off-thread work
server.getAsyncScheduler().runNow(plugin, task -> {
    PlayerData data = database.load(uuid); // never touch the Bukkit API here
});
```

**Rule of thumb:** use `EntityScheduler` (not `RegionScheduler`) for anything that operates on an entity — the region scheduler is tied to a region, an entity moves between regions.

Paper-only alternative annotations are not needed; just prefer these schedulers over `Bukkit.getScheduler()` in all new code.

## Thread Ownership Checks

The API exposes ownership tests, which are available on **regular Paper too** (where they always concern the one main thread):

```java
Bukkit.isOwnedByCurrentRegion(Location location)
Bukkit.isOwnedByCurrentRegion(Location location, int squareRadiusChunks)
Bukkit.isOwnedByCurrentRegion(Block block)
Bukkit.isOwnedByCurrentRegion(Entity entity)
Bukkit.isOwnedByCurrentRegion(World world, Position position)
Bukkit.isOwnedByCurrentRegion(World world, Position position, int squareRadiusChunks)
Bukkit.isOwnedByCurrentRegion(World world, int chunkX, int chunkZ)
Bukkit.isOwnedByCurrentRegion(World world, int chunkX, int chunkZ, int squareRadiusChunks)
Bukkit.isOwnedByCurrentRegion(World world, int minChunkX, int minChunkZ, int maxChunkX, int maxChunkZ)
Bukkit.isGlobalTickThread()
```

Safe pattern for a task that needs to touch a location:

```java
void setBlockSafely(Plugin plugin, Location location, Material material) {
    if (Bukkit.isOwnedByCurrentRegion(location)) {
        location.getBlock().setType(material);          // already on the owning thread
    } else {
        plugin.getServer().getRegionScheduler().execute(plugin, location,
            () -> location.getBlock().setType(material));
    }
}
```

## Thread Context Rules

1. **Commands** for entities/players run on the region owning that entity/player. **Console** commands run on the global region.
2. **Events involving a single entity** (a player breaks/places a block) run on the region owning that entity. Events about an action *on* an entity (e.g. entity damage) run on the region owning the **target** entity.
3. **The async event modifier is deprecated.** All events fired from regions or the global region are considered **synchronous**, even though there is no main thread. Do not rely on `event.isAsynchronous()` to make threading decisions.
4. Regions tick **in parallel, not concurrently**. They do not share data and are not expected to; sharing region-owned data will corrupt it.
5. Rough (not guaranteed) locality: a region owns chunk data within about **8 chunks** of an event's source. Use `isOwnedByCurrentRegion` rather than relying on this.
6. The thread-safety guarantee covers **server** data (entity/chunk/POI). **Your own plugin data is not thread-safe** — normal multithreading rules apply, and events/commands are invoked in parallel. Use concurrent collections where needed, but note that a careless `ConcurrentHashMap` only *hides* threading bugs.

## Broken / Unsupported API on Folia

From Folia's own README ("Current broken API"):

| Area | Status | Workaround |
|------|--------|-----------|
| `Entity#teleport(Location)` | **Never coming back** | Use `Entity#teleportAsync(Location)` |
| Scoreboard API (`Bukkit.getScoreboardManager()`, teams, objectives) | **All considered broken** (global state) | No general workaround; design around it |
| World loading/unloading (`Bukkit.createWorld`, `unloadWorld`) | Broken | Avoid, or run at startup |
| Portals, player respawn, parts of player login API | Broken | — |
| Anything assuming a main thread | Broken | Use the four schedulers |

Also note Folia's planned work: proper async events, world load/unload, and "super aggressive thread checks" that will deliberately **fail hard** on out-of-region access.

## Building Against Folia

```xml
<!-- Maven -->
<repositories>
    <repository>
        <id>papermc</id>
        <url>https://repo.papermc.io/repository/maven-public/</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>dev.folia</groupId>
        <artifactId>folia-api</artifactId>
        <version>[26.2.build,)</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

```kotlin
// Gradle
dependencies {
    compileOnly("dev.folia:folia-api:26.2.build.7-beta")
}
```

For NMS access with Folia, paperweight-userdev offers a Folia dev bundle:

```kotlin
dependencies {
    paperweight.foliaDevBundle("26.2.build.7-beta")
}
```

### Supporting Paper, Folia and Purpur from one JAR

Because the four schedulers and `isOwnedByCurrentRegion` exist on **Paper** as well, the recommended strategy is:

1. Compile against **`paper-api`** (not `folia-api`) using only APIs that exist in both — this keeps the JAR loadable on Paper, Folia and Purpur.
2. Use only the four Paper schedulers, never `Bukkit.getScheduler()`.
3. Replace `entity.teleport(loc)` with `entity.teleportAsync(loc)`.
4. Add `folia-supported: true` to `plugin.yml` **last**, once the plugin genuinely is region-safe.
5. Only compile against `folia-api` when you need an API that truly does not exist on Paper.

## Server Configuration Notes

- Folia wants **at least 16 cores** (cores, not threads) and benefits most from server types that spread players out (skyblock, SMP) with a sizeable player count.
- Pre-generate worlds to cut chunk-system worker demand.
- Tick threads are configured under the global config key `threaded-regions.threads`.
- Do not allocate more than **80%** of available cores across netty I/O, chunk-system I/O, chunk-system workers, GC threads (`-XX:ConcGCThreads`, *not* `-XX:ParallelGCThreads`) and tick threads — plugins and the server spawn unpredicted threads too.

## Migration Checklist (Paper → Folia)

- [ ] Replace every `Bukkit.getScheduler()` call with the appropriate Paper scheduler
- [ ] Replace `entity.teleport(...)` with `entity.teleportAsync(...)`
- [ ] Audit every block/entity/inventory mutation for region ownership (`isOwnedByCurrentRegion`)
- [ ] Remove or isolate scoreboard usage
- [ ] Remove `Bukkit.isPrimaryThread()` assumptions
- [ ] Make plugin-held state thread-safe (or confine it to one region)
- [ ] Stop registering `AsyncChatEvent`/async-event ordering assumptions; treat events as synchronous
- [ ] Avoid runtime world creation/unloading, portals and respawn hooks
- [ ] Test with many spread-out players, not one
- [ ] Add `folia-supported: true` only after all of the above
