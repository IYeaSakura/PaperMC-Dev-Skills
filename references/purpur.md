# Purpur Reference

**Purpur** (PurpurMC) is a **drop-in replacement for Paper**, built from Paper with a large set of opt-in gameplay and configurability patches. It is **not** a Folia fork: it inherits Paper's single-threaded core, so regionised-threading rules do **not** apply to Purpur.

- Repository: https://github.com/PurpurMC/Purpur (default branch `ver/26.2`)
- Docs: https://purpurmc.org/docs/purpur/ · Config: https://purpurmc.org/docs/purpur/configuration/ · Permissions: https://purpurmc.org/docs/purpur/permissions/
- Javadoc: https://purpurmc.org/javadoc/
- Downloads: https://purpurmc.org/downloads · API: `https://api.purpurmc.org/v2/purpur/`
- Discord: https://purpurmc.org/discord

## The Single Most Important Fact for Plugin Authors

> "We set everything we change to the default behaviors. **If you don't edit anything in `purpur.yml`, running this JAR is no different than running Paper.**"

And, from Purpur's FAQ:

> "Do CraftBukkit/Spigot/Paper plugins work on Purpur? **Yes.** The only time there's incompatibility is due to authors hard-coding support for CraftBukkit/Spigot, ignoring the existence of Paper and its forks."

So for most plugins Purpur needs **no code changes at all** — it is one more Paper-compatible target to test against. The risk area is the opposite direction: plugins that *assume* vanilla/Paper game mechanics can break on a server where an admin has enabled a Purpur toggle (rideable mobs, modified block behaviour, changed attributes, …).

## Status & Versions (2026-09-17)

Purpur adopted Paper's post-26.1 versioning — `<mcversion>.build.<n>-<channel>` — with its own build counter. Note the pre-release channel is called **`experimental`**, not `alpha`:

| Purpur line | Latest build | Channel | Notes |
|------------|--------------|---------|-------|
| 26.1.2 | 2592 | **stable** | Previous line |
| **26.2** | **2633** | **stable** | Current recommended target |
| 26.3 | 2637 | experimental | Not for production |

Branch layout: `ver/26.2` and `ver/26.3` (older branches are protected tags like `ver/1.21.11`).

### Downloads API

```bash
# Versions available (metadata.current is the latest)
curl https://api.purpurmc.org/v2/purpur/

# Builds for a version
curl https://api.purpurmc.org/v2/purpur/26.2

# Latest build metadata for a version
curl https://api.purpurmc.org/v2/purpur/26.2/latest

# Download latest
https://api.purpurmc.org/v2/purpur/26.2/latest/download
```

The `/v2/purpur/<version>/latest` response also carries the upstream Paper commit Purpur was built from — useful when a user reports "works on Paper but not Purpur".

## Build Setup

Purpur publishes its own API artifact, which **includes everything from Paper, Pufferfish, Spigot and Bukkit**:

```xml
<!-- Maven -->
<repositories>
    <repository>
        <id>purpur</id>
        <name>Purpur Maven Repo</name>
        <url>https://repo.purpurmc.org/snapshots</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>org.purpurmc.purpur</groupId>
        <artifactId>purpur-api</artifactId>
        <version>26.2.build.2633-stable</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

```kotlin
// Gradle
repositories {
    maven("https://repo.purpurmc.org/snapshots")
}

dependencies {
    compileOnly("org.purpurmc.purpur:purpur-api:26.2.build.2633-stable")
}
```

Metadata: `https://repo.purpurmc.org/snapshots/org/purpurmc/purpur/purpur-api/maven-metadata.xml` (its `<latest>`/`<release>` point at the newest *experimental* build, so parse the list for `-stable`).

### Should you compile against `purpur-api`?

| Goal | Depend on |
|------|-----------|
| Plugin that must also run on Paper/Spigot | **`paper-api`** — keep Purpur-only API behind reflection/optional dependency |
| Plugin that requires Purpur-only API | `purpur-api` (it contains Paper's API, so no second dependency is needed) |

`api-version` and `folia-supported` semantics are unchanged — Purpur inherits Paper's plugin loading rules, so use the same `api-version: '26.2'` value (see [version-matrix.md](version-matrix.md)).

## Purpur API Extensions

All Purpur-specific API lives under `org.purpurmc.purpur.*`.

### Packages

| Package | Contents |
|---------|----------|
| `org.purpurmc.purpur.entity` | `StoredEntity<T extends Entity>` — represents an entity stored in a block |
| `org.purpurmc.purpur.language` | `Language` — translates translation keys |
| `org.purpurmc.purpur.util.permissions` | Purpur default permissions |
| `org.purpurmc.purpur.event` and subpackages | Purpur events (below) |

### Events — `org.purpurmc.purpur.event`

| Event | Fired when |
|-------|-----------|
| `ExecuteCommandEvent` | Someone runs a command |
| `PlayerAFKEvent` | A player goes AFK (Purpur's AFK feature) |
| `PlayerSetSpawnerTypeWithEggEvent` | A spawner type is set with a spawn egg |
| `PlayerSetTrialSpawnerTypeWithEggEvent` | A trial spawner type is set with a spawn egg |
| `PreBlockExplodeEvent` | Before a block's explosion is processed |

### Events — `org.purpurmc.purpur.event.entity`

| Event | Fired when |
|-------|-----------|
| `BeeFoundFlowerEvent` | A bee targets a flower |
| `BeeStartedPollinatingEvent` / `BeeStopPollinatingEvent` | A bee starts/stops pollinating |
| `GoatRamEntityEvent` | A goat rams an entity |
| `LlamaJoinCaravanEvent` / `LlamaLeaveCaravanEvent` | A llama joins/leaves a caravan |
| `PreEntityExplodeEvent` | Before an entity's explosion is processed |
| `RidableMoveEvent` | A ridable mob moves with a rider |
| `RidableSpacebarEvent` | A rider presses spacebar on a ridable mob |

### Events — `org.purpurmc.purpur.event.inventory`

| Event | Fired when |
|-------|-----------|
| `AnvilTakeResultEvent` | A player takes the result out of an anvil |
| `AnvilUpdateResultEvent` | Anvil slots change, updating the result slot |
| `GrindstoneTakeResultEvent` | A player takes the result out of a grindstone |

### Events — `org.purpurmc.purpur.event.player`

| Event | Fired when |
|-------|-----------|
| `PlayerBookTooLargeEvent` | A player tries to bypass book limitations |

### Detecting Purpur

Purpur's Rebrand patch adds an official brand id to `ServerBuildInfo`:

```java
// constant added by Purpur:
Key BRAND_PURPUR_ID = Key.key("purpurmc", "purpur");
```

If you compile against **`purpur-api`**, use the constant directly:

```java
if (ServerBuildInfo.buildInfo().isBrandCompatible(ServerBuildInfo.BRAND_PURPUR_ID)) { … }
```

If you compile against **`paper-api`** (recommended for multi-server plugins), that constant does not exist, so construct the key yourself or fall back to the brand name:

```java
private static boolean isPurpur() {
    // Same id Purpur's patch defines; safe to build on Paper too.
    if (ServerBuildInfo.buildInfo().isBrandCompatible(Key.key("purpurmc", "purpur"))) {
        return true;
    }
    // Optional secondary check; the brand id above is the reliable one.
    return Bukkit.getServer().getName().equalsIgnoreCase("Purpur");
}
```

`ServerBuildInfo` also gives you `brandId()`, `brandName()`, `minecraftVersionId()`, `buildNumber()` and `gitCommit()`, which are handy in bug reports:

```java
getLogger().info("Running on " + ServerBuildInfo.buildInfo().brandName()
    + " " + ServerBuildInfo.buildInfo().minecraftVersionId()
    + " build " + ServerBuildInfo.buildInfo().buildNumber());
```

### Using Purpur API safely

Make Purpur an optional dependency and guard the calls, so one JAR still runs on Paper:

```yaml
# plugin.yml
softdepend: [Purpur]
```

Because a `NoClassDefFoundError` from a Purpur-only import will take the whole plugin down on Paper, put Purpur-only code in its own class and load it only after the brand check:

```java
// PurpurHooks.java — only ever loaded on Purpur
final class PurpurHooks {
    static void register(JavaPlugin plugin) {
        plugin.getServer().getPluginManager().registerEvents(new PurpurBeeListener(), plugin);
    }
}

// onEnable
if (isPurpur()) {
    try {
        PurpurHooks.register(this);
    } catch (NoClassDefFoundError e) {
        getLogger().warning("Purpur API missing despite Purpur brand: " + e.getMessage());
    }
}
```

## `purpur.yml` Configuration

Purpur's features are configured in `purpur.yml`, with a global section and a `world-settings` section (defaults plus per-world overrides).

### Global settings (`settings.` root)

A representative selection — **every** option is documented at https://purpurmc.org/docs/purpur/configuration/:

- Commands & display: `command.uptime.*`, `command.gamemode.requires-specific-permission`, `command.tpsbar.*`, `command.rambar.*`, `command.compass.*`, `command.hide-hidden-players-from-entity-selector`, `messages.*` (AFK broadcasts, sleep messages, death messages, command output)
- AFK: `afk-broadcast-use-display-name`, `afk-tab-list-prefix`, `afk-tab-list-suffix`
- Mechanics: `allow-water-placement-in-the-end`, `use-alternate-keepalive`, `tps-catchup`, `server-mod-name`, `fix-projectile-looting-transfer`, `blast-resistance-overrides`, `clamp-attributes`, `limit-armor`, `username-valid-characters`, `lagging-threshold`, `disable-give-dropping`, `player-deaths-always-show-item`, `generate-end-void-rings`, `bee-count-payload`
- Commands toggles: `register-minecraft-debug-commands`, `register-minecraft-disabled-commands`, `startup-commands`
- Network: `network.kick-for-out-of-order-chat`, `network.upnp-port-forwarding`, `network.max-joins-per-second`
- Logging: `logger.suppress-init-legacy-material-errors`, `suppress-ignored-advancement-warnings`, `suppress-unrecognized-recipe-errors`, `suppress-setblock-in-far-chunk-errors`, `suppress-library-loader`
- Enchanting: `enchantment.allow-looting-on-shears`, `allow-unsafe-enchant-command`, `clamp-levels`, `anvil.allow-inapplicable-enchants`, `allow-incompatible-enchants`, `allow-higher-enchants-levels`, `replace-incompatible-enchants`
- `food-properties`, `entity.enderman.short-height`
- **Blocks (global)**: `blocks.barrel.rows`, `beehive.max-bees-inside`, `grindstone.ignored-enchants` / `remove-attributes` / `remove-name-and-lore`, `ender_chest.persist-hidden-rows` / `six-rows` / `use-permissions-for-rows`, `crying_obsidian.valid-for-portal-frame`, `twisting_vines` / `weeping_vines` / `cave_vines` / `kelp` `max-growth-age`, `anvil.cumulative-cost`, `snow.smooth-accumulation-step`, `leaves.instant-decay`, `lightning_rod.range`, `magma-block.reverse-bubble-column-flow`, `soul-sand.reverse-bubble-column-flow`
- **Broadcasts**: `broadcasts.advancement.only-broadcast-to-affected-player`, `broadcasts.death.only-broadcast-to-affected-player`

### World settings (`world-settings.<world>.`)

Per-world overrides grouped under `hunger`, `settings`, `blocks`, `mobs`:

- `hunger.starvation-damage`
- `settings.entity`, `settings.shared-random`
- **Blocks**: `anvil` (mini-message, colors, iron-ingot repair cost, obsidian damage), `azalea`/`flowering_azalea` growth chance, `beacon` (tinted-glass effects, per-level effect ranges), `bed` (`explode`, `explode-on-villager-sleep`, power/fire/effect), `blue_ice`/`packed_ice` (mob spawns, snow formation), `cactus` (breaks from solid neighbours, bone-mealable), `campfire.lit-when-placed`, `cauldron.fill-chances.*`, `chest.open-with-solid-block-on-top`, `composter.sneak-to-bulk-process`, `conduit` (ring blocks, effect distance, mob damage), `coral.die-outside-water`, `dispenser` (cursed armor slots, place anvils), `door.requires-redstone`, `dragon_egg.teleport`, `enchantment-table.lapis-persists`, `end-crystal` (cramming, baseless/base explosion settings), `farmland` (moisture, alpha farmland, mob-griefing override, trampling controls, trample height), `furnace.use-lava-from-underneath`, `lava` (infinite sources, nether/non-nether speed), `magma-block` (damage while sneaking, frost-walker), `nether_wart`/`sugar_cane` bone-mealable, `observer.disable-clock`, `piston.block-push-limit`, `powder_snow.mob-griefing-override`, `powered-rail.activation-range`, `respawn_anchor` explosion settings, `sculk_shrieker.can-summon-default`, `shulker_box.allow-oversized-stacks`, `sign.allow-colors`, `slab.break-individual-slabs-when-sneaking`, `spawner` (redstone deactivation, `fix-MC-238526`), `sponge` (absorbs lava, absorption area/radius), `stonecutter.damage`, `turtle_egg` (break sources, crack chance, trampling), `water.infinite-required-sources`
- **Mobs**: for most entity types, Purpur exposes `ridable`, `controllable`, `ridable-in-water`, `ridable-max-y`, `takes-damage-from-water`, `no-gravity`, per-mob `attributes.*` (`max_health`, `scale`, `movement_speed`, `follow_range`, `knockback_resistance`, `armor`, `armor_toughness`, `attack_knockback`, `flying_speed`), `always-drop-exp`, `breeding.cooldown-in-ticks`, `breeding.offspring`, and mob-specific toggles (e.g. `bee.can-work-at-night`, `bee.can-work-in-rain`, `bee.dies-after-sting`, `allay.can-pick-up-loot`)

### Implications for plugins

- **Rideable/controllable mobs** are a headline Purpur feature. If your plugin handles mounts, vehicles or `PlayerInteractEntityEvent`, expect mobs that are rideable only on Purpur. `RidableMoveEvent` / `RidableSpacebarEvent` let you hook it.
- **Modified block behaviour** (`farmland.disable-trampling`, `leaves.instant-decay`, `snow.smooth-accumulation-step`, `lava.speed`, `sponge.absorbs-lava`, custom `lightning_rod.range`, …) changes timings your plugin may depend on. Don't hard-code vanilla assumptions; read state from the server.
- **Attribute overrides and clamps** (`clamp-attributes`, per-mob `attributes.*`, `limit-armor`) mean `AttributeInstance#getBaseValue()` may not match vanilla values. Never assume vanilla defaults.
- **Extra commands** (`/tpsbar`, `/rambar`, `/compass`, `/afk`, `/ping`, `/uptime`, `/demo`, `/credits`) can collide with your plugin's command names — check `plugin.yml` aliases before shipping.
- **Permissions**: Purpur adds its own nodes, and per its FAQ some only apply when the matching `purpur.yml` feature is enabled. See https://purpurmc.org/docs/purpur/permissions/.

## Purpur Extras & Packs

Two adjacent projects are useful when an admin wants a feature you'd otherwise be asked to code:

- **PurpurExtras** — a companion plugin: https://purpurmc.org/docs/purpurextras/ (`/purpurextras` commands, its own config)
- **PurpurPacks** — a set of datapacks: https://purpurmc.org/docs/purpurpacks/

## Testing Checklist (Paper plugin on Purpur)

- [ ] Plugin loads with no `purpur.yml` changes (it should behave identically to Paper)
- [ ] Plugin still loads/behaves when Purpur features are enabled: rideable mobs, modified blocks, attribute overrides, extra commands
- [ ] No command-name collisions with Purpur's built-in commands
- [ ] If you use Purpur API: it is optional (`softdepend: [Purpur]`), brand-guarded, and Paper still loads the plugin
- [ ] Platform detection uses `isBrandCompatible(Key.key("purpurmc", "purpur"))` rather than string-matching a display name
- [ ] If reporting a Purpur-only bug, include the upstream Paper commit from `/v2/purpur/<version>/latest`

## Reporting Bugs

- Purpur issues: https://github.com/PurpurMC/Purpur/issues/new
- Feature requests: https://github.com/PurpurMC/Purpur/discussions/new
- Purpur asks that niche features stay in plugins/datapacks rather than the server.
