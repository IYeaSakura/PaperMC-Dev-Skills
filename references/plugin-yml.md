# plugin.yml and paper-plugin.yml Reference

## Table of Contents
1. [plugin.yml (Bukkit Format)](#pluginyml-bukkit-format)
2. [paper-plugin.yml (Paper Native Format)](#paper-plugin-yml)
3. [Critical Rules](#critical-rules)
4. [Choosing Between Formats](#choosing-format)
5. [Common Mistakes](#common-mistakes)

---

## plugin.yml (Bukkit Format)

Place in `src/main/resources/plugin.yml`. This is the standard format compatible with all Bukkit-based servers.

### Complete Example

```yaml
name: YourPlugin
version: '${project.version}'
main: com.yourname.yourplugin.YourPlugin
description: A PaperMC 26.1.2 plugin
author: YourName
authors: [YourName, CoAuthor]
website: https://example.com
api-version: '26.1.2'
load: POSTWORLD

prefix: YP

libraries:
  - com.google.guava:guava:30.1.1-jre
  - com.google.code.gson:gson:2.8.6

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
| `api-version` | Yes | **MUST** be `'26.1.2'` for PaperMC 26.1.2. Controls legacy compat. |
| `description` | No | Short description shown in `/plugins` and info commands. |
| `author` | No | Primary author. |
| `authors` | No | List of authors `[A, B]`. |
| `contributors` | No | Non-author contributors. |
| `website` | No | Plugin website URL. |
| `load` | No | `STARTUP` (before worlds load) or `POSTWORLD` (default, after). |
| `prefix` | No | Log prefix. Defaults to `name`. |
| `libraries` | No | Maven Central deps auto-downloaded by server. |
| `depend` | No | Hard dependencies. Plugin fails to load if missing. |
| `softdepend` | No | Soft dependencies. Load after if present, but not required. |
| `loadbefore` | No | Ensure this plugin loads before listed plugins. |

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

---

## paper-plugin.yml (Paper Native Format)

The newer Paper-native format. **Does NOT replace plugin.yml** — you can include both in the same JAR. If both exist, Paper uses `paper-plugin.yml`.

### Key Differences from plugin.yml

1. **Dependency format is structured** (not flat lists)
2. **Commands are NOT declared in YAML** — register via `JavaPlugin.registerCommand()` in code
3. Supports **bootstrappers** and **loaders** for advanced classpath setup
4. Supports `api-version: '26.1.2'`

### Complete Example

```yaml
name: Paper-Test-Plugin
version: '1.0'
main: io.papermc.testplugin.TestPlugin
description: Paper Test Plugin
author: PaperMC
api-version: '26.1.2'
load: STARTUP
bootstrapper: io.papermc.testplugin.TestPluginBootstrap
loader: io.papermc.testplugin.TestPluginLoader
defaultPerm: FALSE

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

Commands are **NOT** declared in `paper-plugin.yml`. Register them in code:

```java
@Override
public void onEnable() {
    // Register commands programmatically
    getServer().getCommandMap().register("yourplugin", new PluginCommand("yourcmd", this) {
        @Override
        public boolean execute(CommandSender sender, String label, String[] args) {
            // command logic
            return true;
        }
    });
}
```

Or use the Paper command API with Brigadier:

```java
import io.papermc.paper.command.brigadier.Commands;
import com.mojang.brigadier.builder.LiteralArgumentBuilder;

LiteralArgumentBuilder<CommandSourceStack> builder = Commands.literal("yourcmd")
    .requires(source -> source.getSender().hasPermission("yourplugin.use"))
    .executes(ctx -> {
        ctx.getSource().getSender().sendMessage("Hello!");
        return 1;
    });

getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    event.registrar().register(builder.build());
});
```

---

## Critical Rules

### 1. api-version is MANDATORY

Without `api-version`, the server enables Legacy Material Support:
```
[STDERR] CraftLegacy Initializing Legacy Material Support.
Unless you have legacy plugins and/or data this is a bug!
```

Consequences:
- Slower startup (seconds wasted on legacy mapping)
- Inconsistent Material API behavior
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

### 4. Version Placeholder

With Maven resource filtering:
```yaml
version: '${project.version}'
```

This is replaced during build with the actual version from `pom.xml`.

---

## Choosing Format

| Factor | plugin.yml | paper-plugin.yml |
|--------|-----------|------------------|
| Spigot compatibility | Yes | No (Paper only) |
| Command declaration in YAML | Yes | No (code only) |
| Advanced dependency control | No | Yes |
| Bootstrapper/loader support | No | Yes |
| Simplicity | Simpler | More complex |

**Recommendation**: Use `plugin.yml` for most plugins. It is simpler and works across Bukkit/Spigot/Paper. Use `paper-plugin.yml` only when you need its advanced features (structured dependencies, bootstrappers) and Paper-only support is acceptable.

---

## Common Mistakes

1. **Forgetting `api-version`** → Legacy Material Support warning
2. **Space in `name`** → Plugin fails to load
3. **`main` class typo** → `ClassNotFoundException` on startup
4. **Wrong `api-version`** → Use `'26.1.2'` for PaperMC 26.1.2, not `'1.21'`
5. **`version` not quoted** → YAML may parse `1.0` as float. Use `'1.0'` or `"1.0"`
6. **`plugin.yml` not in resources** → Must be at `src/main/resources/plugin.yml`
