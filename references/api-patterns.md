# API Patterns Reference

## Table of Contents
1. [Event System](#event-system)
2. [Command System](#command-system)
3. [Scheduler](#scheduler)
4. [GUI / Inventory](#gui-inventory)
5. [Items](#items)
6. [Entities](#entities)
7. [Worlds](#worlds)
8. [Players](#players)
9. [Configuration](#configuration)
10. [Particles & Sounds](#particles-sounds)
11. [Custom Events](#custom-events)

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
ProjectileHitEvent, ItemSpawnEvent, ItemDespawnEvent
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

---

## Command System

### Basic CommandExecutor

```java
public class MainCommand implements CommandExecutor {
    @Override
    public boolean onCommand(CommandSender sender, Command command,
                            String label, String[] args) {
        if (!(sender instanceof Player player)) {
            sender.sendMessage("Players only!");
            return true;
        }
        if (args.length == 0) {
            player.sendMessage("Usage: /cmd <sub>");
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
            sender.sendMessage("Commands: " + subs.keySet());
            return true;
        }
        SubCommand sub = subs.get(args[0].toLowerCase());
        if (sub == null) { sender.sendMessage("Unknown"); return true; }
        if (!sender.hasPermission(sub.getPermission())) {
            sender.sendMessage("No permission"); return true;
        }
        return sub.execute(sender, Arrays.copyOfRange(args, 1, args.length));
    }
}
```

### Paper Brigadier Commands (Advanced)

```java
import io.papermc.paper.command.brigadier.Commands;
import com.mojang.brigadier.builder.LiteralArgumentBuilder;
import com.mojang.brigadier.builder.RequiredArgumentBuilder;
import io.papermc.paper.command.brigadier.argument.ArgumentTypes;
import io.papermc.paper.plugin.lifecycle.event.LifecycleEventManager;
import io.papermc.paper.plugin.lifecycle.event.types.LifecycleEvents;

LiteralArgumentBuilder<CommandSourceStack> cmd = Commands.literal("myplugin")
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
                }))));

getLifecycleManager().registerEventHandler(LifecycleEvents.COMMANDS, event -> {
    event.registrar().register(cmd.build());
});
```

---

## Scheduler

### Sync Tasks (Main Thread)

```java
// Delayed task (20 ticks = 1 second)
Bukkit.getScheduler().runTaskLater(plugin, () -> {
    player.sendMessage("Delayed!");
}, 20L);

// Repeating task
BukkitTask task = Bukkit.getScheduler().runTaskTimer(plugin, () -> {
    // runs every second
}, 0L, 20L);

// Cancel task
task.cancel();
// Or cancel all plugin tasks
Bukkit.getScheduler().cancelTasks(plugin);
```

### Async Tasks

```java
Bukkit.getScheduler().runTaskAsynchronously(plugin, () -> {
    // DB query, file I/O, HTTP request
});

Bukkit.getScheduler().runTaskTimerAsynchronously(plugin, () -> {
    // Repeating async task (auto-save, etc.)
}, 0L, 6000L); // every 5 minutes
```

### Async -> Sync Bridge

```java
// NEVER call Bukkit API from async threads directly
Bukkit.getScheduler().runTaskAsynchronously(plugin, () -> {
    PlayerData data = database.load(uuid); // async-safe

    // Switch back to main thread for Bukkit API
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.teleport(data.getHomeLocation()); // main thread only
        player.sendMessage("Teleported!");
    });
});
```

### BukkitRunnable

```java
new BukkitRunnable() {
    int count = 0;
    @Override
    public void run() {
        count++;
        if (count >= 10) this.cancel();
    }
}.runTaskTimer(plugin, 0L, 20L);
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
meta.setCustomModelData(1001);

// Store custom identifier
meta.getPersistentDataContainer().set(
    new NamespacedKey(plugin, "item_id"), PersistentDataType.STRING, "legendary_sword"
);

item.setItemMeta(meta);
```

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

### Custom Mob with Goals (Paper API)

```java
// Paper provides MobGoal API for modifying entity AI
target.getWorld().spawn(target.getLocation(), Zombie.class, zombie -> {
    zombie.setCustomName("Custom Mob");
    zombie.setAI(true);
    // Configure before it's added to world
});
```

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

---

## Players

```java
// Messages (MiniMessage format recommended)
player.sendMessage("Legacy color: " + ChatColor.GREEN + "Hello");
// Or with MiniMessage (Adventure API):
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

// Teleport
player.teleport(new Location(world, x, y, z, yaw, pitch));

// Inventory
player.getInventory().addItem(item);
player.getInventory().removeItem(item);
player.openInventory(gui.getInventory());
player.closeInventory();

// Effects
player.addPotionEffect(new PotionEffect(PotionEffectType.SPEED, 200, 1));

// XP
player.giveExp(100);
player.setLevel(player.getLevel() + 1);

// Game mode
player.setGameMode(GameMode.SURVIVAL);

// Health/food
player.setHealth(20);
player.setFoodLevel(20);
player.setSaturation(5);

// Permissions
player.hasPermission("node");
player.isOp();
```

---

## Configuration

### Default Config

```java
@Override
public void onEnable() {
    saveDefaultConfig();     // Copies config.yml from jar if not exists
    saveResource("shops.yml", false); // Copy without overwriting
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
world.spawnParticle(Particle.DUST, location, 1,
    new Particle.DustOptions(Color.RED, 1.0f));

// Sounds
player.playSound(location, Sound.ENTITY_PLAYER_LEVELUP, 1.0f, 1.0f);
world.playSound(location, Sound.BLOCK_NOTE_BLOCK_PLING, 1.0f, 2.0f);

// For all nearby players
world.playSound(location, Sound.ENTITY_GENERIC_EXPLODE, SoundCategory.BLOCKS, 1.0f, 1.0f);
```

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
