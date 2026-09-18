# Project Setup Reference

Target: **Paper 26.2 stable (Minecraft Java 26.2), Java 25.** Substitute the version you actually target (`26.1.2`, `26.3`, …) and the corresponding `api-version`.

This page sets up a **Paper** project, which is the right default: a JAR compiled against `paper-api` (using only APIs shared with the forks) also loads on **Purpur** and, with `folia-supported: true`, on **Folia**. Fork-specific setup is at the end of this file.

## Table of Contents
1. [Maven pom.xml (Complete)](#maven-pomxml)
2. [Gradle Kotlin DSL (Recommended)](#gradle-kotlin-dsl)
3. [Version Pinning Rules](#version-pinning)
4. [Project Structure](#project-structure)
5. [Build Commands](#build-commands)
6. [Local Test Server](#local-test-server)
7. [Dependencies Guide](#dependencies-guide)
8. [Targeting Folia or Purpur](#fork-targets)

---

## Maven pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.yourname</groupId>
    <artifactId>your-plugin</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>YourPlugin</name>
    <description>A PaperMC 26.2 plugin</description>

    <properties>
        <!-- release = enforce the Java 25 API level (source/target alone do not) -->
        <maven.compiler.release>25</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- Pin an exact -stable build. Paper labels Maven ranges "Discouraged". -->
        <paper.api.version>26.2.build.124-stable</paper.api.version>
    </properties>

    <repositories>
        <repository>
            <id>papermc</id>
            <url>https://repo.papermc.io/repository/maven-public/</url>
        </repository>
    </repositories>

    <dependencies>
        <!-- Paper API (provided = the server already has it) -->
        <dependency>
            <groupId>io.papermc.paper</groupId>
            <artifactId>paper-api</artifactId>
            <version>${paper.api.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- Optional: Caffeine caching -->
        <dependency>
            <groupId>com.github.ben-manes.caffeine</groupId>
            <artifactId>caffeine</artifactId>
            <version>3.1.8</version>
            <scope>compile</scope>
        </dependency>

        <!-- Optional: MySQL connector -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <version>8.4.0</version>
            <scope>compile</scope>
        </dependency>

        <!-- Optional: HikariCP connection pool -->
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>5.1.0</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <release>25</release>
                    <compilerArgs>
                        <arg>-parameters</arg>
                    </compilerArgs>
                </configuration>
            </plugin>

            <!-- Shade plugin (bundle dependencies into the jar) -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.6.0</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals><goal>shade</goal></goals>
                        <configuration>
                            <minimizeJar>true</minimizeJar>
                            <relocations>
                                <!-- Relocate to avoid conflicts with other plugins -->
                                <relocation>
                                    <pattern>com.zaxxer</pattern>
                                    <shadedPattern>com.yourname.libs.hikari</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>com.github.benmanes.caffeine</pattern>
                                    <shadedPattern>com.yourname.libs.caffeine</shadedPattern>
                                </relocation>
                            </relocations>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                        <exclude>**/module-info.class</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>

        <resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>true</filtering>
            </resource>
        </resources>
    </build>
</project>
```

### Mapping Namespace Manifest Entry

For `plugin.yml` plugins on 26.x, declaring Mojang mappings skips the one-time runtime remap and keeps you compatible across minor updates. If you use Maven, add it with `maven-jar-plugin`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.4.2</version>
    <configuration>
        <archive>
            <manifestEntries>
                <paperweight-mappings-namespace>mojang</paperweight-mappings-namespace>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

(Paper plugins using `paper-plugin.yml` are assumed Mojang-mapped and do not need this.)

---

## Gradle Kotlin DSL

Gradle is Paper's own build system and the only one the docs cover first-class, so prefer it for new projects.

```kotlin
plugins {
    java
    id("com.gradleup.shadow") version "8.3.5"   // or com.github.johnrengelman.shadow for older Gradle
}

group = "com.yourname"
version = "1.0.0-SNAPSHOT"

repositories {
    mavenCentral()
    maven("https://repo.papermc.io/repository/maven-public/")
}

dependencies {
    // Exact build (reproducible). The documented loose form is "26.2.build.+";
    // keep the literal `build` token — "26.2.+" could resolve to another patch line.
    compileOnly("io.papermc.paper:paper-api:26.2.build.124-stable")
    implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
}

java {
    toolchain.languageVersion.set(JavaLanguageVersion.of(25))
}

tasks.withType<JavaCompile> {
    options.release.set(25)
    options.compilerArgs.add("-parameters")
}

// Tell the server this JAR is Mojang-mapped (skip the legacy remap).
tasks.jar {
    manifest {
        attributes["paperweight-mappings-namespace"] = "mojang"
    }
}
tasks.named<ShadowJar>("shadowJar") {
    manifest {
        attributes["paperweight-mappings-namespace"] = "mojang"
    }
}
```

### Running a test server from Gradle

```kotlin
plugins {
    id("xyz.jpenilla.run-paper") version "3.0.2"
}

tasks {
    runServer {
        minecraftVersion("26.2")
    }
}
```

### NMS / internal access

Use **paperweight-userdev** instead of a plain `paper-api` dependency. Current version: **2.0.0-beta.23**.

```kotlin
plugins {
    id("io.papermc.paperweight.userdev") version "2.0.0-beta.23"
}

dependencies {
    paperweight.paperDevBundle("26.2.build.124-stable")
    // NOTE: remove the paper-api dependency — the dev bundle already contains it.
}
```

See [version-matrix.md](version-matrix.md) for the mappings, `reobfJar` and dev-bundle details.

---

## Version Pinning

| Approach | Example | Verdict |
|----------|---------|---------|
| Exact stable build | `26.2.build.124-stable` | **Preferred** — reproducible |
| Gradle loose build | `26.2.build.+` | Documented, acceptable for plugins that must track fixes |
| Maven range | `[26.2.build,)` | Paper labels this **"Maven (Discouraged)"** |
| Guessed version | `26.2` or `1.26.2` | **Wrong** — these are not Maven artifact versions |

To find the newest stable build:

- `https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml` (parse the version list — `<latest>`/`<release>` may point at an `-alpha`)
- `https://fill.papermc.io/v3/projects/paper/versions/26.2/builds` (each entry has a `channel`)

The old `https://api.papermc.io/v2/...` downloads API is **sunset** (HTTP 410); use the v3 API.

---

## Project Structure

```
your-plugin/
├── pom.xml                          # Maven config (or build.gradle.kts for Gradle)
└── src/
    └── main/
        ├── java/
        │   └── com/
        │       └── yourname/
        │           └── yourplugin/
        │               ├── YourPlugin.java          # Main class (extends JavaPlugin)
        │               ├── api/                     # Public API for other plugins
        │               ├── command/                 # Command executors + tab completers
        │               ├── listener/                # Event listeners
        │               ├── config/                  # Config management
        │               ├── gui/                     # Inventory GUIs
        │               ├── database/                # Database access
        │               ├── model/                   # Data models/records
        │               ├── cache/                   # Caching layer
        │               ├── task/                    # Scheduled tasks
        │               └── util/                    # Utility classes
        └── resources/
            ├── plugin.yml           # Plugin metadata (REQUIRED)
            ├── config.yml           # Default config
            └── messages.yml         # Messages config
```

---

## Build Commands

```bash
# Maven build (creates shaded jar with dependencies)
mvn clean package
# Output: target/your-plugin-1.0.0-SNAPSHOT.jar

# Gradle build
./gradlew build
# Output: build/libs/your-plugin-1.0.0-SNAPSHOT-all.jar
```

---

## Local Test Server

```bash
# 1. Download Paper 26.2 from https://papermc.io/downloads/paper
#    The jar is named like paper-26.2-124.jar
# 2. Create a start script

# run.sh (Linux/macOS)
#!/bin/bash
java -Xms4G -Xmx4G -jar paper-26.2-124.jar nogui

# run.bat (Windows)
@echo off
java -Xms4G -Xmx4G -jar paper-26.2-124.jar nogui
pause

# 3. First run generates eula.txt — set eula=true
# 4. Place the plugin jar in plugins/ and restart
```

**Java 25 is required to run the server.** Verify with `java -version` before blaming the plugin.

---

## Dependencies Guide

| Dependency | Purpose | Scope |
|-----------|---------|-------|
| `paper-api` | Core API | `provided` / `compileOnly` |
| `caffeine` | High-performance cache | `compile` (shade) |
| `mysql-connector-j` | MySQL database | `compile` (shade) |
| `HikariCP` | Connection pooling | `compile` (shade) |
| `sqlite-jdbc` | SQLite (usually unnecessary — the driver ships with the server) | `compile` (shade) |

Notes:

- **Do not bundle `paper-api`** — the server provides it.
- Always shade **and relocate** third-party libraries to avoid conflicts with other plugins.
- `plugin.yml` `libraries:` can download Maven Central deps at runtime instead of shading, but Paper's docs flag it as currently against Maven Central's TOS; use shading for anything you ship.

---

## Targeting Folia or Purpur

The most portable plugin compiles against `paper-api` and stays inside the API shared by all three servers. Only switch the dependency when you genuinely need fork-only API.

### Folia

```kotlin
dependencies {
    // Swap the coordinate; Folia is in the PaperMC repo, group dev.folia
    compileOnly("dev.folia:folia-api:26.2.build.7-beta")
}

// Optional: NMS access for Folia
dependencies {
    paperweight.foliaDevBundle("26.2.build.7-beta")
}
```

```yaml
# plugin.yml — REQUIRED or Folia will not load the plugin at all
folia-supported: true
```

Folia 26.2 is currently **beta**, not stable — its last fully stable line is 26.1.2. Prefer compiling against `paper-api` and using the shared schedulers unless you need a Folia-only type. Details: [folia.md](folia.md).

### Purpur

```kotlin
repositories {
    maven("https://repo.purpurmc.org/snapshots")
}

dependencies {
    // purpur-api includes Paper + Pufferfish + Spigot + Bukkit API
    compileOnly("org.purpurmc.purpur:purpur-api:26.2.build.2633-stable")
}
```

```yaml
# plugin.yml — Purpur is optional so the same JAR still loads on Paper
softdepend: [Purpur]
```

Prefer `paper-api` plus a brand check if you only need Purpur API on some servers; import Purpur types in an isolated class so Paper never has to resolve them. Details: [purpur.md](purpur.md).

### Multi-server test matrix

If the plugin claims fork support, test it on each server:

1. It loads (Folia is the strict one — no `folia-supported` means no load).
2. Schedulers fire the expected number of times (region merging/splitting can change Folia's timing).
3. Nothing throws thread-ownership / "out of region" errors on Folia.
4. Purpur with `purpur.yml` left at defaults behaves exactly like Paper — then retest with the toggles you care about enabled.
