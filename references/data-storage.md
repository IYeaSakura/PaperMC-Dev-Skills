# Data Persistence Reference

## Table of Contents
1. [SQLite (Recommended for Small Plugins)](#sqlite)
2. [MySQL with HikariCP (Large Servers)](#mysql)
3. [Async Database Operations](#async-operations)
4. [Player Data Cache Pattern](#cache-pattern)
5. [Caffeine Cache](#caffeine-cache)
6. [YAML File Storage](#yaml-storage)

---

## SQLite

Paper ships an SQLite JDBC driver on the classpath, so you normally need no extra dependency. If you compile against it explicitly (or run on a platform without it), shade `org.xerial:sqlite-jdbc` and relocate it as usual. Store the database file in the plugin data folder.

### Connection Manager

```java
public class SQLiteManager {
    private final JavaPlugin plugin;
    private Connection connection;

    public SQLiteManager(JavaPlugin plugin) { this.plugin = plugin; }

    public void connect() {
        try {
            File dataFolder = plugin.getDataFolder();
            if (!dataFolder.exists()) dataFolder.mkdirs();

            String url = "jdbc:sqlite:" + new File(dataFolder, "data.db").getAbsolutePath();
            connection = DriverManager.getConnection(url);
            createTables();
            plugin.getLogger().info("SQLite connected.");
        } catch (SQLException e) {
            plugin.getLogger().severe("SQLite connection failed: " + e.getMessage());
        }
    }

    private void createTables() throws SQLException {
        String sql = """
            CREATE TABLE IF NOT EXISTS players (
                uuid TEXT PRIMARY KEY,
                name TEXT NOT NULL,
                coins INTEGER DEFAULT 0,
                level INTEGER DEFAULT 1,
                last_login TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            );
            """;
        try (Statement stmt = connection.createStatement()) {
            stmt.execute(sql);
        }
    }

    public void disconnect() {
        try {
            if (connection != null && !connection.isClosed()) connection.close();
        } catch (SQLException e) {
            plugin.getLogger().warning("Error closing SQLite: " + e.getMessage());
        }
    }

    public Connection getConnection() { return connection; }
}
```

### CRUD Operations

```java
public class PlayerDataDao {
    private final Connection conn;

    public PlayerDataDao(Connection conn) { this.conn = conn; }

    public void insert(PlayerData data) throws SQLException {
        String sql = "INSERT OR REPLACE INTO players (uuid, name, coins, level) VALUES (?, ?, ?, ?)";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, data.uuid().toString());
            ps.setString(2, data.name());
            ps.setInt(3, data.coins());
            ps.setInt(4, data.level());
            ps.executeUpdate();
        }
    }

    public PlayerData findByUuid(UUID uuid) throws SQLException {
        String sql = "SELECT * FROM players WHERE uuid = ?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, uuid.toString());
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    return new PlayerData(
                        UUID.fromString(rs.getString("uuid")),
                        rs.getString("name"),
                        rs.getInt("coins"),
                        rs.getInt("level")
                    );
                }
            }
        }
        return null;
    }

    public void delete(UUID uuid) throws SQLException {
        String sql = "DELETE FROM players WHERE uuid = ?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, uuid.toString());
            ps.executeUpdate();
        }
    }

    // Batch insert
    public void batchInsert(List<PlayerData> players) throws SQLException {
        String sql = "INSERT INTO players (uuid, name, coins, level) VALUES (?, ?, ?, ?)";
        conn.setAutoCommit(false);
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            for (PlayerData data : players) {
                ps.setString(1, data.uuid().toString());
                ps.setString(2, data.name());
                ps.setInt(3, data.coins());
                ps.setInt(4, data.level());
                ps.addBatch();
            }
            ps.executeBatch();
            conn.commit();
        } catch (SQLException e) {
            conn.rollback();
            throw e;
        } finally {
            conn.setAutoCommit(true);
        }
    }
}
```

### Record Model

```java
public record PlayerData(UUID uuid, String name, int coins, int level) {
    public PlayerData {
        if (coins < 0) throw new IllegalArgumentException("Coins cannot be negative");
    }
}
```

---

## MySQL

Add dependencies to pom.xml: `mysql-connector-j` and `HikariCP`.

### HikariCP Manager

```java
public class MySQLManager {
    private HikariDataSource dataSource;
    private final JavaPlugin plugin;

    public MySQLManager(JavaPlugin plugin) { this.plugin = plugin; }

    public void connect(String host, int port, String database,
                       String username, String password) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(String.format(
            "jdbc:mysql://%s:%d/%s?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true",
            host, port, database));
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setLeakDetectionThreshold(60000);

        // MySQL performance tuning
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");

        dataSource = new HikariDataSource(config);
        plugin.getLogger().info("MySQL connected.");
    }

    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }

    public void disconnect() {
        if (dataSource != null && !dataSource.isClosed()) dataSource.close();
    }
}
```

---

## Async Operations

> On Paper 26.x prefer the Paper schedulers — `getAsyncScheduler()` for off-thread work and
> `getGlobalRegionScheduler()` / `getRegionScheduler()` to come back to the owning thread.
> They behave the same on Paper and Folia; `Bukkit.getScheduler()` still works on plain Paper
> but is not Folia-compatible.

### Pattern: Async Query + Main Thread Callback

```java
public void loadPlayerDataAsync(UUID uuid, Consumer<PlayerData> callback) {
    plugin.getServer().getAsyncScheduler().runNow(plugin, task -> {
        PlayerData data;
        try { data = dao.findByUuid(uuid); }
        catch (SQLException e) { data = null; }

        final PlayerData result = data;
        plugin.getServer().getGlobalRegionScheduler().run(plugin, scheduled ->
            callback.accept(result));
    });
}

// Usage
loadPlayerDataAsync(player.getUniqueId(), data -> {
    if (data != null) player.sendMessage(Component.text("Coins: " + data.coins()));
});
```

The older `Bukkit.getScheduler()` equivalent (still valid on Paper, not on Folia):

```java
Bukkit.getScheduler().runTaskAsynchronously(plugin, () -> {
    PlayerData data = dao.findByUuid(uuid);
    Bukkit.getScheduler().runTask(plugin, () -> callback.accept(data));
});
```

### Periodic Auto-Save

```java
// Paper scheduler: fixed rate in milliseconds, off-thread
plugin.getServer().getAsyncScheduler().runAtFixedRate(plugin, task -> {
    manager.saveAll();
}, 0L, 5L, TimeUnit.MINUTES);
```

### CompletableFuture Pattern

```java
public CompletableFuture<PlayerData> getPlayerDataAsync(UUID uuid) {
    return CompletableFuture.supplyAsync(() -> {
        try { return dao.findByUuid(uuid); }
        catch (SQLException e) { return null; }
    });
}

// Usage
getPlayerDataAsync(player.getUniqueId()).thenAccept(data -> {
    Bukkit.getScheduler().runTask(plugin, () -> {
        player.sendMessage("Loaded: " + data.name());
    });
});
```

---

## Cache Pattern

### Simple ConcurrentHashMap Cache

```java
public class PlayerDataManager {
    private final Map<UUID, PlayerData> cache = new ConcurrentHashMap<>();
    private final PlayerDataDao dao;

    public PlayerDataManager(PlayerDataDao dao) { this.dao = dao; }

    public PlayerData getData(Player player) {
        return cache.computeIfAbsent(player.getUniqueId(), uuid -> {
            try {
                PlayerData data = dao.findByUuid(uuid);
                return data != null ? data : new PlayerData(uuid, player.getName(), 0, 1);
            } catch (SQLException e) {
                return new PlayerData(uuid, player.getName(), 0, 1);
            }
        });
    }

    public void saveData(UUID uuid) {
        PlayerData data = cache.get(uuid);
        if (data != null) {
            try { dao.insert(data); }
            catch (SQLException e) { /* log error */ }
        }
    }

    public void saveAll() {
        for (Map.Entry<UUID, PlayerData> entry : cache.entrySet()) {
            try { dao.insert(entry.getValue()); }
            catch (SQLException e) { /* log error */ }
        }
    }

    public void unload(UUID uuid) {
        saveData(uuid);
        cache.remove(uuid);
    }

    public void unloadAll() {
        saveAll();
        cache.clear();
    }
}
```

---

## Caffeine Cache

Add dependency: `com.github.ben-manes.caffeine:caffeine:3.1.8`

### Loading Cache (Auto-load missing entries)

```java
public class PlayerCache {
    private final LoadingCache<UUID, PlayerData> cache;

    public PlayerCache(PlayerDataDao dao) {
        this.cache = Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .refreshAfterWrite(5, TimeUnit.MINUTES)
            .recordStats()
            .build(uuid -> {
                try { return dao.findByUuid(uuid); }
                catch (SQLException e) { return null; }
            });
    }

    public PlayerData get(UUID uuid) { return cache.get(uuid); }
    public void invalidate(UUID uuid) { cache.invalidate(uuid); }
    public void invalidateAll() { cache.invalidateAll(); }
    public void put(UUID uuid, PlayerData data) { cache.put(uuid, data); }
}
```

### Manual Cache

```java
Cache<UUID, PlayerData> cache = Caffeine.newBuilder()
    .maximumSize(5_000)
    .expireAfterAccess(15, TimeUnit.MINUTES)
    .build();

// Get (may return null)
PlayerData data = cache.getIfPresent(uuid);

// Get or compute
data = cache.get(uuid, uuid -> loadFromDatabase(uuid));

// Put
cache.put(uuid, data);

// Invalidate
cache.invalidate(uuid);
```

---

## YAML Storage

For simple data that doesn't need SQL, use Bukkit's YAML configuration:

```java
// Player data stored per-file
public class YamlPlayerData {
    private final JavaPlugin plugin;

    public void savePlayer(Player player) {
        File file = new File(plugin.getDataFolder() + "/playerdata",
            player.getUniqueId() + ".yml");
        FileConfiguration config = YamlConfiguration.loadConfiguration(file);
        config.set("name", player.getName());
        config.set("coins", getCoins(player));
        config.set("level", getLevel(player));
        config.set("last-login", System.currentTimeMillis());
        try { config.save(file); }
        catch (IOException e) { plugin.getLogger().warning("Save failed"); }
    }

    public void loadPlayer(Player player) {
        File file = new File(plugin.getDataFolder() + "/playerdata",
            player.getUniqueId() + ".yml");
        if (!file.exists()) return;
        FileConfiguration config = YamlConfiguration.loadConfiguration(file);
        int coins = config.getInt("coins", 0);
        int level = config.getInt("level", 1);
        // apply to player...
    }
}
```

**YAML storage is suitable for:** config data, small player counts, simple key-value data
**YAML storage is NOT suitable for:** large datasets, concurrent writes, relational data, search/filter operations
