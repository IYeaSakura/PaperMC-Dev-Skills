# API Patterns Reference

Targets Paper 26.x (verified against `paper-api 26.2.build.124-stable`). Patterns here use **only APIs shared by Paper, Purpur and Folia** unless a section says otherwise — see [folia.md](folia.md) and [purpur.md](purpur.md) for fork-specific API.

## Table of Contents
1. [Event System](#event-system)
2. [Command System](#command-system)
3. [Scheduler](#scheduler)
4. [GUI / Inventory](#gui-inventory)
5. [Items](#items)
6. [Entities](#entities)
7. [Worlds](#worlds)
8. [Players](#players)
9. [Text & Adventure 5](#text-adventure)
10. [Configuration](#configuration)
11. [Particles & Sounds](#particles-sounds)
12. [Custom Events](#custom-events)
13. [26.x Migration Notes](#migration-notes)
14. [Writing One Plugin for Paper + Folia + Purpur](#multi-target)

---

## Event System

### Basic Listener

```java
public class PlayerListener implements Listener {

    @EventHandler
    public void onPlayerJoin(PlayerJoinEvent event) {
        Player player = event.getPlayer();
        event.joinMessage(Component.text("Welcome, " + player.getName() + "!"));
    }

    @EventHandler
    public void onPlayerQuit(PlayerQuitEvent event) {
        event.quitMessage(Component.text(event.getPlayer().getName() + " left."));
    }
}
```

### Event Priority

```java
@EventHandler(priority = EventPriority.HIGH)
public void onEvent(SomeEvent event) {
    // LOWEST -> LOW -> NORMAL -> HIGH -> HIGHEST -> MONITOR
    // LOWEST: read original state first
    // HIGH/HIGHEST: modify event outcome
    // MONITOR: read-only observation, never modify
}
```

### Ignore Cancelled

```java
@EventHandler(priority = EventPriority.NORMAL, ignoreCancelled = true)
public void onBlockBreak(BlockBreakEvent event) {
    // Skipped if another plugin already cancelled the event
}
```

### Common Events

**Player events:**
```
PlayerJoinEvent, PlayerQuitEvent, PlayerMoveEvent, PlayerInteractEvent,
PlayerInteractEntityEvent, AsyncChatEvent, PlayerCommandPreprocessEvent,
PlayerDeathEvent, PlayerRespawnEvent, PlayerTeleportEvent, PlayerItemHeldEvent,
PlayerDropItemEvent, PlayerPickupItemEvent, PlayerToggleSneakEvent,
PlayerToggleSprintEvent, PlayerItemConsumeEvent, PlayerExpChangeEvent,
PlayerLevelChangeEvent, PlayerBedEnterEvent, PlayerBedLeaveEvent
```

**Block/World events:**
```
BlockBreakEvent, BlockPlaceEvent, BlockDamageEvent, BlockGrowEvent,
BlockSpreadEvent, BlockFormEvent, BlockFadeEvent, BlockBurnEvent,
BlockExplodeEvent, ChunkLoadEvent, ChunkUnloadEvent, WorldLoadEvent,
WorldUnloadEvent, StructureGrowEvent
```

**Entity events:**
```
EntityDamageEvent, EntityDamageByEntityEvent, EntityDeathEvent,
EntitySpawnEvent, EntityTargetEvent, EntityTeleportEvent, EntityExplodeEvent,
EntityRegainHealthEvent, CreatureSpawnEvent, ProjectileLaunchEvent,
ProjectileHitEvent, ItemSpawnEvent, ItemDespawnEvent,
EntityIgniteEvent (new in 26.2, parent of CreeperIgniteEvent)
```

**Inventory events:**
```
InventoryClickEvent, InventoryDragEvent, InventoryOpenEvent,
InventoryCloseEvent, InventoryMoveItemEvent, PrepareItemCraftEvent,
CraftItemEvent, FurnaceBurnEvent, FurnaceSmeltEvent
```

### Register Listeners

```java
@Override
public void onEnable() {
    PluginManager pm = getServer().getPluginManager();
    pm.registerEvents(new PlayerListener(), this);
    pm.registerEvents(new BlockListener(), this);
    pm.registerEvents(new GUIListener(this), this);
}
```

### 26.2: Rescuing PersistentDataContainer Data from Removed Block Entities

Paper 26.2 removed the bed block entity, and future versions will remove more. When the data fixer drops a block entity, Paper fires `AsyncServerDataFixerRemoveBlockEntityEvent` so you can salvage the PDC:

```java
@EventHandler
public void onBlockEntityRemoved(AsyncServerDataFixerRemoveBlockEntityEvent event) {
    if (!event.getBlockEntityType().equals(Key.key("minecraft", "bed"))) return;

    PersistentDataContainerView pdc = event.getPersistentDataContainerView();
    String myData = pdc.get(new NamespacedKey(this, "my_key"), PersistentDataType.STRING);
    if (myData == null) return;

    Key worldKey = event.getWorldKey();
    BlockPosition pos = event.getBlockPosition();

    // WARNING: this fires during chunk loading, on a worker thread OR the main
    // thread. Do not block here — hand the work to your own executor and only
    // touch the Bukkit API back on the main thread.
    getServer().getAsyncScheduler().runNow(this, task ->
        getLogger().info("Rescued " + myData + " from bed at " + pos + " in " + worldKey));
}
```

Relevant methods: `getBlockEntityType()`, `getWorldKey()`, `getBlockPosition()`, `getPersistentDataContainerView()` (an immutable `PersistentDataContainerView`).

Notes:

- The event is **not** `Cancellable` — you cannot prevent the removal, only salvage the data.
- Despite the `Async` prefix it fires during chunk loading, so it may run on a chunk-loading worker thread **or** the main thread. The Javadoc explicitly says heavy/blocking work is strongly discouraged because the main thread may be blocked waiting on those workers — hence the "enqueue and process elsewhere" shape above.
- `Server.getWorld(Key)` is the intended way to resolve `getWorldKey()`.
- Paper's own PDC guide does not yet mention the bed change, so don't rely on it for bed-PDC questions.

---

## Command System

### Basic CommandExecutor

```java
public class MainCommand implements CommandExecutor {
    @Override
    public boolean onCommand(CommandSender sender, Command command,
                            String label, String[] args) {
        if (!(sender instanceof Player player)) {
            sender.sendMessage(Component.text("Players only!"));
            return true;
        }
        if (args.length == 0) {
            player.sendMessage(Component.text("Usage: /cmd <sub>"));
            return true;
        }
        // handle subcommands
        return true; // return false to show usage from plugin.yml
    }
}
```

### TabCompleter

```java
public class MainTabCompleter implements TabCompleter {
    private static final List<String> SUBS = List.of("reload", "give", "help");

    @Override
    public List<String> onTabComplete(CommandSender sender, Command command,
                                      String alias, String[] args) {
        if (args.length == 1) {
            return SUBS.stream()
                .filter(s -> s.startsWith(args[0].toLowerCase()))
                .toList();
        }
        return List.of();
    }
}
```

### SubCommand Pattern

```java
public interface SubCommand {
    boolean execute(CommandSender sender, String[] args);
    String getPermission();
    List<String> tabComplete(CommandSender sender, String[] args);
}

public class MainCommand implements CommandExecutor {
    private final Map<String, SubCommand> subs = new HashMap<>();

    public MainCommand() {
        subs.put("reload", new ReloadSubCommand());
        subs.put("give", new GiveSubCommand());
    }

    @Override
    public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
        if (args.length == 0) {
            sender.sendMessage(Component.text("Commands: " + subs.keySet()));
            return true;
        }
        SubCommand sub = subs.get(args[0].toLowerCase());
        if (sub == null) { sender.sendMessage(Component.text("Unknown")); return true; }
        if (!sender.hasPermission(sub.getPermission())) {
            sender.sendMessage(Component.text("No permission")); return true;
        }
        return sub.execute(sender, Arrays.copyOfRange(args, 1, args.length));
    }
}
```

### Paper Brigadier Commands (Recommended on 26.x)

```java
import io.papermc.paper.command.brigadier.Commands;
import io.papermc.paper.command.brigadier.CommandSourceStack;
import io.papermc.paper.command.brigadier.argument.ArgumentTypes;
import io.papermc.paper.command.brigadier.argument.resolvers.selector.PlayerSelectorArgumentResolver;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;

@Override
public void onEnable() {
    getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
        event.registrar().register(
            Commands.literal("myplugin")
                .requires(src -> src.getSender().hasPermission("myplugin.use"))
                .then(Commands.literal("give")
                    .then(Commands.argument("player", ArgumentTypes.player())
                        .then(Commands.argument("item", ArgumentTypes.itemStack())
                            .executes(ctx -> {
                                Player target = ctx.getArgument("player", PlayerSelectorArgumentResolver.class)
                                    .resolve(ctx.getSource()).getFirst();
                                ItemStack item = ctx.getArgument("item", ItemStack.class);
                                target.getInventory().addItem(item);
                                return 1;
                            }))))
                .build()
        );
    });
}
```

Registering commands this way works with both `plugin.yml` and `paper-plugin.yml`. The older `com.destroystokyo.paper.brigadier.BukkitBrigadierCommand` and `io.papermc.paper.brigadier.PaperBrigadier` helpers are deprecated for removal.

---

## Scheduler

> On **Folia** these four schedulers are not a preference — they are the only correct option, because there is no main thread. See [folia.md](folia.md).

### Paper Schedulers (Preferred on 26.x)

```java
Server server = getServer();

// Server-wide work on the main thread
server.getGlobalRegionScheduler().run(plugin, task -> {
    // main-thread work
});

// Delayed / repeating, main thread
server.getGlobalRegionScheduler().runDelayed(plugin, task -> { /* ... */ }, 20L);
server.getGlobalRegionScheduler().runAtFixedRate(plugin, task -> { /* ... */ }, 0L, 20L);

// Location/chunk-owned work (the Folia-safe way to touch blocks and entities)
server.getRegionScheduler().run(plugin, location, task -> {
    location.getBlock().setType(Material.STONE);
});
server.getRegionScheduler().runDelayed(plugin, location, task -> { /* ... */ }, 20L);

// Off-thread work: DB, file I/O, HTTP
server.getAsyncScheduler().runNow(plugin, task -> {
    // never touch the Bukkit API here
});
server.getAsyncScheduler().runAtFixedRate(plugin, task -> { /* ... */ }, 0L, 6000L, TimeUnit.MILLISECONDS);
```

| Operation | Required thread |
|-----------|-----------------|
| Modify blocks | Owning region thread (main on Paper) |
| Operate entities / player inventory | Owning region thread (main on Paper) |
| Database queries, file I/O, HTTP | Async |

### BukkitRunnable / Bukkit Scheduler

The classic `Bukkit.getScheduler()` API still works on regular Paper and is fine for simple plugins — but it is **not** Folia-compatible, so prefer the Paper schedulers for new code:

```java
// Delayed task (20 ticks = 1 second)
Bukkit.getScheduler().runTaskLater(plugin, () -> {
    player.sendMessage(Component.text("Delayed!"));
}, 20L);

// Repeating task
BukkitTask task = Bukkit.getScheduler().runTaskTimer(plugin, () -> {
    // runs every second
}, 0L, 20L);

// Cancel
task.cancel();
Bukkit.getScheduler().cancelTasks(plugin);
```

### Async → Sync Bridge

```java
getServer().getAsyncScheduler().runNow(plugin, task -> {
    PlayerData data = database.load(uuid);            // async-safe

    getServer().getGlobalRegionScheduler().run(plugin, scheduledTask -> {
        player.sendMessage(Component.text("Loaded: " + data.coins()));
    });
});
```

### Folia Detection

```java
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

## GUI / Inventory

### InventoryHolder Pattern

```java
public class ShopGUI implements InventoryHolder {
    private final Inventory inv;
    private final Player player;

    public ShopGUI(Player player) {
        this.player = player;
        this.inv = Bukkit.createInventory(this, 54, Component.text("Shop"));
        fillBorders();
        setItems();
    }

    private void fillBorders() {
        ItemStack border = createItem(Material.GRAY_STAINED_GLASS_PANE, " ");
        for (int i = 0; i < 54; i++) {
            if (i < 9 || i >= 45 || i % 9 == 0 || i % 9 == 8) {
                inv.setItem(i, border);
            }
        }
    }

    private ItemStack createItem(Material mat, String name, String... lore) {
        ItemStack item = new ItemStack(mat);
        ItemMeta meta = item.getItemMeta();
        meta.displayName(Component.text(name));
        if (lore.length > 0) {
            meta.lore(Arrays.stream(lore).map(Component::text).toList());
        }
        item.setItemMeta(meta);
        return item;
    }

    @Override public Inventory getInventory() { return inv; }
    public void open() { player.openInventory(inv); }
}
```

### Click Handler

```java
public class GUIListener implements Listener {
    @EventHandler
    public void onClick(InventoryClickEvent event) {
        if (!(event.getInventory().getHolder() instanceof ShopGUI)) return;
        event.setCancelled(true);

        Player player = (Player) event.getWhoClicked();
        int slot = event.getRawSlot();

        switch (slot) {
            case 20 -> openBuyPage(player);
            case 24 -> openSellPage(player);
            case 49 -> player.closeInventory();
        }
    }

    @EventHandler
    public void onDrag(InventoryDragEvent event) {
        if (event.getInventory().getHolder() instanceof ShopGUI) {
            event.setCancelled(true);
        }
    }
}
```

Remember `event.getRawSlot()` can be in the player's own inventory — always check the holder, and cancel `InventoryDragEvent` as well as clicks.

### PersistentDataContainer for GUI Actions

```java
public class GUIItemUtil {
    public static ItemStack tagAction(ItemStack item, String action, JavaPlugin plugin) {
        ItemMeta meta = item.getItemMeta();
        meta.getPersistentDataContainer().set(
            new NamespacedKey(plugin, "gui_action"), PersistentDataType.STRING, action);
        item.setItemMeta(meta);
        return item;
    }

    public static String getAction(ItemStack item, JavaPlugin plugin) {
        if (item == null || !item.hasItemMeta()) return null;
        return item.getItemMeta().getPersistentDataContainer().get(
            new NamespacedKey(plugin, "gui_action"), PersistentDataType.STRING);
    }
}
```

---

## Items

### Create Custom Item

```java
ItemStack item = new ItemStack(Material.DIAMOND_SWORD);
ItemMeta meta = item.getItemMeta();
meta.displayName(Component.text("Legendary Sword").decorate(TextDecoration.BOLD));
meta.lore(List.of(
    Component.text("A sword of legends").color(NamedTextColor.GRAY)
));
meta.addEnchant(Enchantment.SHARPNESS, 5, true);
meta.addItemFlags(ItemFlag.HIDE_ENCHANTS, ItemFlag.HIDE_ATTRIBUTES);
meta.setUnbreakable(true);

// Store custom identifier
meta.getPersistentDataContainer().set(
    new NamespacedKey(plugin, "item_id"), PersistentDataType.STRING, "legendary_sword"
);

item.setItemMeta(meta);
```

### Prefer Data Components over Legacy Custom Model Data

Modern Paper exposes the vanilla item data-component system. Where 1.20.5+ plugins used `setCustomModelData(int)`, prefer the component API:

```java
import io.papermc.paper.datacomponent.DataComponentTypes;
import io.papermc.paper.datacomponent.item.CustomModelData;

item.setData(DataComponentTypes.CUSTOM_MODEL_DATA, CustomModelData.customModelData().addString("legendary"));
```

Also note `Enchantment` and friends are registry-keyed: use `Registry` lookups rather than `Enchantment.getByName(...)`.

### Check Item in Inventory

```java
public boolean hasItem(Player player, Material material, int amount) {
    int count = 0;
    for (ItemStack stack : player.getInventory().getContents()) {
        if (stack != null && stack.getType() == material) {
            count += stack.getAmount();
        }
    }
    return count >= amount;
}

public void removeItems(Player player, Material material, int amount) {
    player.getInventory().removeItem(new ItemStack(material, amount));
}
```

### Read Custom Data from Item

```java
public String getCustomId(ItemStack item, JavaPlugin plugin) {
    if (item == null || !item.hasItemMeta()) return null;
    return item.getItemMeta().getPersistentDataContainer().get(
        new NamespacedKey(plugin, "item_id"), PersistentDataType.STRING);
}
```

---

## Entities

### Spawn and Configure

```java
Location loc = player.getLocation();
Zombie zombie = (Zombie) loc.getWorld().spawnEntity(loc, EntityType.ZOMBIE);

zombie.setCustomName("Elite Zombie");
zombie.setCustomNameVisible(true);
zombie.setHealth(100);
zombie.getAttribute(Attribute.MAX_HEALTH).setBaseValue(100);

zombie.getEquipment().setHelmet(new ItemStack(Material.DIAMOND_HELMET));
zombie.getEquipment().setHelmetDropChance(0.5f);

zombie.addPotionEffect(new PotionEffect(PotionEffectType.SPEED, Integer.MAX_VALUE, 2));
zombie.setTarget(player);
```

### Spawn with a Consumer (pre-add configuration)

```java
target.getWorld().spawn(target.getLocation(), Zombie.class, zombie -> {
    zombie.setCustomName("Custom Mob");
    zombie.setAI(true);
    // Configure before the entity is added to the world
});
```

### 26.2: Cube Mobs and `AbstractCubeMob`

`MagmaCube` no longer extends `Slime`. Both now implement `org.bukkit.entity.AbstractCubeMob` (subinterfaces: `Slime`, `MagmaCube`, `SulfurCube`), which exposes:

```java
interface AbstractCubeMob extends Creature {
    int getSize();
    void setSize(int size);
    boolean canWander();
    void setWander(boolean wander);
}
```

Update `instanceof Slime` checks to `instanceof AbstractCubeMob`, and note the consequences:

- `SlimeSplitEvent#getEntity()` now returns `AbstractCubeMob`. The erased method descriptor changed, so a plugin compiled against 26.1 gets `NoSuchMethodError` on 26.2 — **recompile**.
- Spawning was broken until 26.2 build #30 (`IllegalArgumentException: Cannot spawn an entity for org.bukkit.entity.AbstractCubeMob`, [issue #13978](https://github.com/PaperMC/Paper/issues/13978)); spawn concrete types (`EntityType.SLIME`, `EntityType.MAGMA_CUBE`) and use a build with the fix.

```java
@EventHandler
public void onSlimeSplit(SlimeSplitEvent event) {
    AbstractCubeMob cube = event.getEntity();
    cube.setSize(1);                 // slime / magma cube / sulfur cube uniformly
    // ...
}
```

### Attribute Modifiers Are Keyed

```java
// 26.x: modifiers are identified by Key, not UUID
AttributeInstance instance = zombie.getAttribute(Attribute.MAX_HEALTH);
instance.addModifier(new AttributeModifier(
    new NamespacedKey(plugin, "elite_bonus"), 20.0, AttributeModifier.Operation.ADD_NUMBER));
instance.removeModifier(new NamespacedKey(plugin, "elite_bonus"));
```

The `UUID`-taking `AttributeModifier` constructors and `getModifier(UUID)`/`removeModifier(UUID)` are deprecated.

### Teleport Flags

`TeleportFlag.EntityState.RETAIN_PASSENGERS`, `RETAIN_OPEN_INVENTORY` and `RETAIN_VEHICLE` are deprecated — passengervehicle retention is default vanilla behavior now, and open inventories must be closed manually.

---

## Worlds

```java
World world = Bukkit.getWorld("world");
world.setTime(6000);
world.setStorm(false);
world.setGameRule(GameRule.KEEP_INVENTORY, true);

WorldBorder border = world.getWorldBorder();
border.setCenter(0, 0);
border.setSize(10000);

// Get all worlds
for (World w : Bukkit.getWorlds()) {
    getLogger().info("World: " + w.getName());
}
```

`World#setSpawnFlags(...)` and `World#getAllowAnimals()` are deprecated in 26.2 (vanilla no longer has an animal-spawn toggle) — drop those calls.

⚠️ **World storage format changed in 26.1.** After a world is upgraded you cannot downgrade it. Back up worlds before first start on 26.x.

---

## Players

```java
// Messages (Adventure Component / MiniMessage recommended)
player.sendMessage(Component.text("Hello " + player.getName(), NamedTextColor.GREEN));
player.sendMessage(MiniMessage.miniMessage().deserialize("<green>Hello <player>",
    Placeholder.component("player", Component.text(player.getName()))));

// Title
player.showTitle(Title.title(
    Component.text("Title"),
    Component.text("Subtitle"),
    Title.Times.times(Duration.ofMillis(500), Duration.ofSeconds(3), Duration.ofMillis(500))
));

// Action bar
player.sendActionBar(Component.text("Action bar message"));

// Sounds
player.playSound(player.getLocation(), Sound.ENTITY_PLAYER_LEVELUP, 1.0f, 1.0f);

// Teleport — async form works on Paper, Purpur and Folia alike
player.teleportAsync(new Location(world, x, y, z, yaw, pitch)).thenAccept(success -> {
    if (!success) getLogger().warning("Teleport failed for " + player.getName());
});

// Inventory
player.getInventory().addItem(item);
player.getInventory().removeItem(item);
player.openInventory(gui.getInventory());
player.closeInventory();

// Effects
player.addPotionEffect(new PotionEffect(PotionEffectType.SPEED, 200, 1));

// XP / level
player.giveExp(100);
player.setLevel(player.getLevel() + 1);

// Game mode / health / food
player.setGameMode(GameMode.SURVIVAL);
player.setHealth(20);
player.setFoodLevel(20);
player.setSaturation(5);

// Permissions
player.hasPermission("node");
player.isOp();
```

`player.sendMessage("string")` still compiles via the legacy overload, but it bypasses component styling — pass a `Component` in new code.

---

## Text & Adventure 5

Paper 26.2 ships **Adventure 5**. Key points for plugin authors:

- **`Component` implementations are sealed.** You can no longer create a custom `Component` class; use `VirtualComponent`.
- **`BuildableComponent` was removed.** Get a builder from `Component#toBuilder` instead.
- **`ClickEvent` is a typed interface** and `ClickEvent.Action` is no longer an enum. `ClickEvent#value`, `ClickEvent#create(Action, String)` and custom-payload string constructors were removed — use the payload-based `create` or the direct factories such as `ClickEvent.openUrl(String)`.
- **Legacy chat-signing API removed**: the `MessageType` enum is gone, and `Audience#sendMessage` overloads taking `Identity`/`Identified` were removed. Send signed messages instead.
- **Boss bar `percent` API removed** — use the progress constants/methods.
- `Component#join`/`replaceText` variants without a `JoinConfiguration`/`TextReplacementConfig` were removed; `TranslationRegistry` → `TranslationStore`; `PlainComponentSerializer` → `PlainTextComponentSerializer`; `JSONComponentConstants` → `ComponentTreeConstants`.
- JSpecify nullness annotations replaced JetBrains annotations, and Adventure now requires **Java 21+** (fine on 26.x).
- `adventure-extra-kotlin` and `adventure-text-serializer-gson-legacy-impl` modules were removed.

Practical text pattern:

```java
private final MiniMessage mm = MiniMessage.miniMessage();

Component msg = mm.deserialize("<gradient:#00ff00:#00aa00>Welcome</gradient> <gray>to the server!");
player.sendMessage(msg);
```

For configurable messages, keep the raw MiniMessage string in `config.yml` and deserialize at send time (or cache per reload).

### Books on 26.2 (BookMeta)

`BookMeta` no longer implements Adventure's `Book`, so the builder and the inherited `Book` setters are gone. `BookMeta` is mutable — set its fields directly:

```java
// 26.1.x (removed in 26.2)
BookMeta built = meta.toBuilder()
    .title(Component.text("Guide"))
    .author(Component.text("Me"))
    .pages(List.of(Component.text("page 1")))
    .build();

// 26.2+
BookMeta meta = (BookMeta) item.getItemMeta();
meta.title(Component.text("Guide"));
meta.author(Component.text("Me"));
meta.pages(List.of(Component.text("page 1"), Component.text("page 2")));
meta.addPages(Component.text("appendix"));
item.setItemMeta(meta);

// Need an Adventure Book (e.g. player.openBook(Book))?
net.kyori.adventure.inventory.Book book = meta.asBook();
```

Available on 26.2: `pages(List<Component>)`, `pages(Component...)`, `pages()`, `page(int, Component)`, `page(int)`, `title(Component)`, `title()`, `author(Component)`, `author()`, `addPages(Component...)`, `asBook()`. Plugins that used the inherited setters compile on 26.1.x and fail at runtime with `NoSuchMethodError` on 26.2 — recompile against the 26.2 API.

Code that used `PlainComponentSerializer` should use `PlainTextComponentSerializer`; `PaperComponents.gsonSerializer()` and friends are terminally deprecated — use the plain Adventure serializers.

---

## Configuration

### Default Config

```java
@Override
public void onEnable() {
    saveDefaultConfig();                 // Copies config.yml from the jar if missing
    saveResource("shops.yml", false);    // Copy without overwriting
}
```

### Read Config

```java
FileConfiguration config = getConfig();

String msg = config.getString("messages.welcome", "Default");
int max = config.getInt("settings.max-players", 100);
double price = config.getDouble("shop.price", 10.0);
boolean enabled = config.getBoolean("features.enabled", true);
List<String> worlds = config.getStringList("allowed-worlds");

// ConfigurationSection
ConfigurationSection items = config.getConfigurationSection("shop.items");
if (items != null) {
    for (String key : items.getKeys(false)) {
        String mat = items.getString(key + ".material");
        int price = items.getInt(key + ".price");
    }
}
```

### Custom Config File Manager

```java
public class CustomConfig {
    private final JavaPlugin plugin;
    private final String fileName;
    private File file;
    private FileConfiguration config;

    public CustomConfig(JavaPlugin plugin, String fileName) {
        this.plugin = plugin;
        this.fileName = fileName;
        setup();
    }

    public void setup() {
        file = new File(plugin.getDataFolder(), fileName);
        if (!file.exists()) {
            file.getParentFile().mkdirs();
            plugin.saveResource(fileName, false);
        }
        config = YamlConfiguration.loadConfiguration(file);
    }

    public FileConfiguration getConfig() { return config; }

    public void save() {
        try { config.save(file); }
        catch (IOException e) { plugin.getLogger().severe("Cannot save " + fileName); }
    }

    public void reload() { config = YamlConfiguration.loadConfiguration(file); }
}
```

---

## Particles & Sounds

```java
// Particles
world.spawnParticle(Particle.HEART, location, 10, 0.5, 0.5, 0.5);
world.spawnParticle(Particle.DUST, location, 1, new Particle.DustOptions(Color.RED, 1.0f));

// Sounds
player.playSound(location, Sound.ENTITY_PLAYER_LEVELUP, 1.0f, 1.0f);
world.playSound(location, Sound.BLOCK_NOTE_BLOCK_PLING, 1.0f, 2.0f);

// For all nearby players
world.playSound(location, Sound.ENTITY_GENERIC_EXPLODE, SoundCategory.BLOCKS, 1.0f, 1.0f);
```

Sound and Particle are registry-backed types; prefer constants/registry lookups over name-based lookups (`Sound.valueOf` is legacy).

---

## Custom Events

```java
public class CustomTradeEvent extends Event implements Cancellable {
    private static final HandlerList handlers = new HandlerList();
    private final Player buyer;
    private final Player seller;
    private final double price;
    private boolean cancelled = false;

    public CustomTradeEvent(Player buyer, Player seller, double price) {
        this.buyer = buyer;
        this.seller = seller;
        this.price = price;
    }

    public Player getBuyer() { return buyer; }
    public Player getSeller() { return seller; }
    public double getPrice() { return price; }

    @Override public boolean isCancelled() { return cancelled; }
    @Override public void setCancelled(boolean cancel) { this.cancelled = cancel; }

    @Override public HandlerList getHandlers() { return handlers; }
    public static HandlerList getHandlerList() { return handlers; }
}
```

Call custom event:
```java
CustomTradeEvent event = new CustomTradeEvent(buyer, seller, price);
Bukkit.getPluginManager().callEvent(event);
if (!event.isCancelled()) {
    // process trade
}
```

For configuration-driven listeners, `@EventHandler` methods work with `registerEvents`; for a listener with constructor state use an instance field (as in `new GUIListener(this)`).

---

## 26.x Migration Notes

Checklist when bringing a plugin forward to 26.x:

1. **Java 25** toolchain and `options.release = 25`.
2. **`api-version`** set to the lowest 26.x you support (`'26.1'` or `'26.2'`).
3. **Use `Component`s**, not `String` messages/names; move to MiniMessage for user-facing text.
4. **Adventure 5** compiles? Fix `ClickEvent`, `BookMeta` and removed `Audience` methods (see [Text & Adventure 5](#text-adventure)).
5. **Beds**: stop reading/writing bed PDCs; handle `AsyncServerDataFixerRemoveBlockEntityEvent`.
6. **Replace** `BlockSoundGroup` → `SoundGroup`, `TargetBlockInfo` → `RayTraceResult`, `PointedDripstone` block data → `Speleothem`, `PinkPetals` block data → `FlowerBed`.
7. **Drop the conversation API**; use `AsyncChatEvent` or `Dialog`.
8. **Profile with spark** instead of Timings.
9. **Schedule with the Paper schedulers** if you want Folia support.
10. **Re-test world-touching code** — the 26.1 world storage format change is irreversible.

---

## Writing One Plugin for Paper + Folia + Purpur

A single JAR can support all three if you stay inside the shared API and handle the two fork-specific concerns (Folia threading, optional Purpur API).

### 1. Detect the platform once

```java
public final class ServerFlavor {
    private static final ServerBuildInfo INFO = ServerBuildInfo.buildInfo();

    public static boolean isFolia() {
        return INFO.isBrandCompatible(Key.key("papermc", "folia"));
    }

    public static boolean isPurpur() {
        return INFO.brandName().equalsIgnoreCase("Purpur");
    }

    public static boolean isPaper() {
        return !isFolia() && !isPurpur();
    }

    private ServerFlavor() {}
}
```

### 2. Never use `Bukkit.getScheduler()` again

The four Paper schedulers behave identically on Paper/Purpur (main thread) and Folia (region thread):

```java
// location-owned work — safe on all three
server.getRegionScheduler().execute(plugin, location, () -> location.getBlock().setType(Material.STONE));

// entity-owned work — follows the entity across Folia regions
entity.getScheduler().run(plugin, task -> entity.setFireTicks(0), null);

// server-wide / global state
server.getGlobalRegionScheduler().execute(plugin, () -> world.setStorm(false));

// off-thread work
server.getAsyncScheduler().runNow(plugin, task -> saveToDatabase());
```

### 3. Replace sync teleports

```java
// Never on Folia; Entity#teleport is permanently unsupported there
entity.teleport(destination);

// Works everywhere
entity.teleportAsync(destination);
```

### 4. Guard optional fork API behind its own class

```java
// PurpurHooks.java — only loaded when isPurpur() is true
final class PurpurHooks {
    static void registerPurpurListeners(JavaPlugin plugin) {
        plugin.getServer().getPluginManager().registerEvents(new PurpurBeeListener(), plugin);
    }
}

// in onEnable
if (ServerFlavor.isPurpur()) {
    try {
        Class.forName("org.purpurmc.purpur.event.entity.BeeFoundFlowerEvent");
        PurpurHooks.registerPurpurListeners(this);   // body references Purpur types
    } catch (ClassNotFoundException | NoClassDefFoundError e) {
        getLogger().warning("Purpur API unavailable; skipping Purpur integration.");
    }
}
```

Keeping the Purpur-typed code in `PurpurHooks` means the JVM never resolves `org.purpurmc.purpur.*` on Paper, so no `NoClassDefFoundError`.

### 5. Declare capabilities in `plugin.yml`

```yaml
api-version: '26.2'
softdepend: [Purpur]        # optional integration, Paper still loads
folia-supported: true      # ONLY after the plugin is genuinely region-safe
```

### 6. Know what you gave up

| If you use | Paper | Purpur | Folia |
|------------|:----:|:------:|:-----:|
| Paper schedulers (above) | ✅ | ✅ | ✅ |
| `teleportAsync` | ✅ | ✅ | ✅ |
| Scoreboard API | ✅ | ✅ | ❌ |
| Runtime world create/unload | ✅ | ✅ | ❌ |
| `Bukkit.getScheduler()` | ✅ | ✅ | ❌ |
| Purpur events | ❌ | ✅ | ❌ |

See [folia.md](folia.md) for the full broken-API list and migration checklist, and [purpur.md](purpur.md) for `purpur.yml` behaviour that can surprise a plugin assuming vanilla mechanics.
