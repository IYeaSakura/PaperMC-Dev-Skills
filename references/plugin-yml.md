# plugin.yml and paper-plugin.yml Reference

## Table of Contents
1. [plugin.yml (Bukkit Format)](#pluginyml-bukkit-format)
2. [paper-plugin.yml (Paper Native Format)](#paper-plugin-yml)
3. [api-version Rules](#api-version-rules)
4. [Critical Rules](#critical-rules)
5. [Choosing Between Formats](#choosing-format)
6. [Common Mistakes](#common-mistakes)

---

## plugin.yml (Bukkit Format)

Place in `src/main/resources/plugin.yml`. This is the standard format compatible with all Bukkit-based servers.

### Complete Example

```yaml
name: YourPlugin
version: '${project.version}'
main: com.yourname.yourplugin.YourPlugin
description: A PaperMC 26.2 plugin
author: YourName
authors: [YourName, CoAuthor]
website: https://example.com
api-version: '26.2'
load: POSTWORLD

prefix: YP

libraries:
  - com.google.guava:guava:33.3.1-jre
  - com.google.code.gson:gson:2.11.0

depend: [Vault, LuckPerms]
softdepend: [PlaceholderAPI, WorldGuard]
loadbefore: [AnotherPlugin]

commands:
  yourplugin:
    description: Main plugin command
    usage: /yourplugin <subcommand>
    aliases: [yp, ypl]
    permission: yourplugin.use
    permission-message: "&cYou don't have permission!"
  shop:
    description: Open shop GUI
    usage: /shop
    permission: yourplugin.shop

permissions:
  yourplugin.use:
    description: Basic use permission
    default: true
  yourplugin.admin:
    description: Admin permission
    default: op
    children:
      yourplugin.use: true
      yourplugin.reload: true
      yourplugin.shop: true
  yourplugin.reload:
    description: Reload config permission
    default: op
  yourplugin.shop:
    description: Access shop
    default: true
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Plugin name. Only `[a-zA-Z0-9_-]+`. No spaces. |
| `version` | Yes | Plugin version. Use `${project.version}` for Maven auto-fill. |
| `main` | Yes | Fully qualified main class name. Must extend `JavaPlugin`. |
| `api-version` | Yes | Minecraft/Paper **API** version, e.g. `'26.2'`. See [api-version Rules](#api-version-rules). |
| `description` | No | Short description shown in `/plugins` and info commands. |
| `author` | No | Primary author. |
| `authors` | No | List of authors `[A, B]`. |
| `contributors` | No | Non-author contributors. |
| `website` | No | Plugin website URL. |
| `load` | No | `STARTUP` (before worlds load) or `POSTWORLD` (default, after). |
| `prefix` | No | Log prefix. Defaults to `name`. |
| `libraries` | No | Maven Central deps auto-downloaded by the server. Paper's docs warn this is currently against Maven Central's TOS — prefer shading. |
| `default-permission` | No | Default for permission nodes without an explicit `default`. |
| `depend` | No | Hard dependencies. Plugin fails to load if missing. |
| `softdepend` | No | Soft dependencies. Load after if present, but not required. |
| `loadbefore` | No | Ensure this plugin loads before the listed plugins. |
| `provides` | No | Declare that this plugin provides another plugin's functionality/alias. |
| `folia-supported` | No | **Required on Folia.** `true` opts the plugin into regionised multithreading; without it Folia refuses to load the plugin. Ignored on Paper/Purpur. See [folia-support](#folia-support). |

### folia-support

Folia loads **only** plugins that explicitly opt in:

```yaml
name: YourPlugin
main: com.yourname.yourplugin.YourPlugin
api-version: '26.2'
folia-supported: true     # Folia will not load the plugin without this
```

The flag is a *declaration*, not a compatibility switch. Folia's README is blunt about it: only set it after you have actually made the plugin region-safe (Paper schedulers instead of `Bukkit.getScheduler()`, `teleportAsync` instead of `teleport`, no scoreboard/world-load assumptions, thread-safe plugin state). Setting it on an unmodified Paper plugin produces silent data corruption rather than a clean error. Full checklist: [folia.md](folia.md).

### permissions

```yaml
permissions:
  yourplugin.node:
    description: "What this permission does"
    default: op          # op / not_op / true / false
    children:
      yourplugin.child: true   # Grant child when parent granted
```

- `default: true` = all players have it
- `default: op` = only ops have it
- `default: false` = nobody has it by default
- `default: not_op` = non-ops have it (rarely used)
- `default-permission:` sets the fallback `default` for nodes that omit it

### commands

```yaml
commands:
  commandname:
    description: "What this command does"
    usage: "/commandname <arg>"
    aliases: [alias1, alias2]
    permission: yourplugin.permission
    permission-message: "&cNo permission!"
```

Note: players only see commands they have permission for (`permission` filters both execution and visibility).

---

## paper-plugin.yml (Paper Native Format)

The newer Paper-native format. **Does NOT replace plugin.yml** — you can include both in the same JAR. If both exist, Paper uses `paper-plugin.yml`.

### Key Differences from plugin.yml

1. **Dependency format is structured** (not flat lists)
2. **Commands are NOT declared in YAML** — register them in code
3. Supports **bootstrappers** and **loaders** for advanced classpath setup
4. Uses the same `api-version` semantics
5. JARs are assumed **Mojang-mapped** (no `paperweight-mappings-namespace` needed)

### Complete Example

```yaml
name: Paper-Test-Plugin
version: '1.0'
main: io.papermc.testplugin.TestPlugin
description: Paper Test Plugin
author: PaperMC
api-version: '26.2'
load: STARTUP
bootstrapper: io.papermc.testplugin.TestPluginBootstrap
loader: io.papermc.testplugin.TestPluginLoader
defaultPerm: FALSE
folia-supported: true

permissions:
  testplugin.use:
    description: Basic permission
    default: true

dependencies:
  server:
    - name: Vault
      required: true
      join-classpath: true
    - name: LuckPerms
      required: false
      join-classpath: true
  bootstrap:
    - name: SomeBootstrapDep
      required: true
```

### Dependency Format

```yaml
# Old (plugin.yml):
depend: [Vault, LuckPerms]
softdepend: [PlaceholderAPI]

# New (paper-plugin.yml):
dependencies:
  server:
    - name: Vault
      required: true          # equivalent to "depend"
      join-classpath: true
    - name: PlaceholderAPI
      required: false         # equivalent to "softdepend"
      join-classpath: true
```

### Commands in paper-plugin.yml

Commands are **NOT** declared in `paper-plugin.yml`. Register them via the lifecycle API — this is the supported path on 26.x:

```java
import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;

@Override
public void onEnable() {
    getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
        event.registrar().register(
            Commands.literal("yourcmd")
                .requires(source -> source.getSender().hasPermission("yourplugin.use"))
                .executes(ctx -> {
                    ctx.getSource().getSender().sendMessage(Component.text("Hello!"));
                    return 1;
                })
                .build()
        );
    });
}
```

The legacy `BukkitBrigadierCommand` / `PaperBrigadier` helpers are deprecated for removal on 26.x.

---

## api-version Rules

`api-version` is the **Minecraft/Paper API version**, never a Paper build id:

```yaml
api-version: '26.2'    # CORRECT for Paper 26.2
api-version: '26.1'    # 26.1 API — loads on 26.1.1/26.1.2 and 26.2
api-version: '1.21'    # legacy 1.21 line only
```

Rules (from `org.bukkit.craftbukkit.util.ApiVersion` and `CraftMagicNumbers#checkSupported`):

1. The value must be `major.minor` or `major.minor.patch` with **numeric** parts. `26.2.0`, `26.2`, `26.1.2` and `1.20.5` are all valid; `26.2.build.124-stable`, `1.26.2`, `latest` throw `IllegalArgumentException`.
2. There is **no allow-list**. The value is only range-checked against the server's API version and the optional `settings.minimum-api` floor in `bukkit.yml` (default `none`).
3. Newer than the server → `InvalidPluginException: Unsupported API version <value>`; the plugin does not load.
4. Older than `settings.minimum-api` → `InvalidPluginException: Plugin API version … is lower than the minimum allowed version`.
5. Omitted → legacy load with `Legacy plugin <name> does not specify an api-version.` and possible Legacy Material Support.
6. The valid range per Paper's docs is **1.13 – latest Paper release**; minor versions (`x.y.z`) are supported from 1.20.5 onward.

**Pick the lowest version that supports every API you call** so the plugin keeps loading across the 26.x line. See [version-matrix.md](version-matrix.md) for the full per-version table.

---

## Critical Rules

### 1. api-version is MANDATORY

Without `api-version`, the server treats the plugin as legacy:
```
[STDERR] CraftLegacy Initializing Legacy Material Support.
Unless you have legacy plugins and/or data this is a bug!
```

Consequences:
- Slower startup (seconds wasted on legacy mapping)
- Inconsistent Material API behavior
- `Material#isLegacy()` values appearing in your own item handling
- Potential conflicts with other plugins

### 2. name Format

```yaml
# VALID
name: YourPlugin
name: your-plugin
name: Your_Plugin_2

# INVALID - will fail to load
name: Your Plugin    # space
name: Your.Plugin    # dot
```

### 3. main Class Requirements

- Must exist in the JAR
- Must extend `org.bukkit.plugin.java.JavaPlugin`
- Must have a public no-arg constructor (default is fine)
- Must be compiled for **Java 25** on 26.x — an older target fails to load

### 4. Version Placeholder

With Maven resource filtering:
```yaml
version: '${project.version}'
```

This is replaced during build with the actual version from `pom.xml`. Quote it — unquoted `1.0` is parsed as a float.

---

## Choosing Format

| Factor | plugin.yml | paper-plugin.yml |
|--------|-----------|------------------|
| Spigot/other-server compatibility | Yes | No (Paper only) |
| Command declaration in YAML | Yes | No (code only) |
| Advanced dependency control | No | Yes |
| Bootstrapper/loader support | No | Yes |
| Mapping assumption | Spigot unless the manifest says otherwise | Mojang |
| Simplicity | Simpler | More complex |

**Recommendation**: use `plugin.yml` for most plugins — it is simpler and works across Bukkit/Spigot/Paper/Purpur/Folia. Use `paper-plugin.yml` only when you need its advanced features (structured dependencies, bootstrappers) and Paper-only support is acceptable. Note that a `paper-plugin.yml`-only plugin will **not** load on Spigot (and products without Paper's new plugin loader), so it narrows your audience. Paper's own docs still describe the Paper plugin format as **experimental**.

---

## Common Mistakes

1. **Forgetting `api-version`** → Legacy Material Support warning
2. **Using a Paper build id as `api-version`** (`26.2.build.124-stable`) → parsing error; use `'26.2'`
3. **Assuming `26.1.2` ≡ `26.2`** → they are different `major.minor.patch` triples; `26.1.2` does not satisfy a 26.2 requirement
4. **Space in `name`** → plugin fails to load
5. **`main` class typo** → `ClassNotFoundException` on startup
6. **`version` not quoted** → YAML may parse `1.0` as float. Use `'1.0'`
7. **`plugin.yml` not in resources** → must be at `src/main/resources/plugin.yml`
8. **Compiling for Java 21 or below** → `UnsupportedClassVersionError` on 26.x
9. **Claiming Folia support without `folia-supported: true`** → the plugin is silently not loaded at all on Folia
10. **Setting `folia-supported: true` on an unmodified Paper plugin** → it loads and then corrupts state; only declare what you have actually made region-safe
11. **Hard-importing `org.purpurmc.purpur.*` without `softdepend: [Purpur]` and a brand check** → `NoClassDefFoundError` takes the plugin down on Paper
