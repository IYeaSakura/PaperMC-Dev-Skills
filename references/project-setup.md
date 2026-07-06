# Project Setup Reference

## Table of Contents
1. [Maven pom.xml (Complete)](#maven-pomxml)
2. [Gradle Kotlin DSL (Alternative)](#gradle-kotlin-dsl)
3. [Project Structure](#project-structure)
4. [Build Commands](#build-commands)
5. [Local Test Server](#local-test-server)
6. [Dependencies Guide](#dependencies-guide)

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
    <description>A PaperMC 26.1.2 plugin</description>

    <properties>
        <maven.compiler.source>25</maven.compiler.source>
        <maven.compiler.target>25</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- Use latest stable build or range -->
        <paper.api.version>26.1.2.build.72-stable</paper.api.version>
    </properties>

    <repositories>
        <repository>
            <id>papermc</id>
            <url>https://repo.papermc.io/repository/maven-public/</url>
        </repository>
    </repositories>

    <dependencies>
        <!-- Paper API (provided = server already has it) -->
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
            <!-- Compiler plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
                <configuration>
                    <source>25</source>
                    <target>25</target>
                    <compilerArgs>
                        <arg>-parameters</arg>
                    </compilerArgs>
                </configuration>
            </plugin>

            <!-- Shade plugin (bundle dependencies into jar) -->
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

### Using Version Ranges (for auto-updates)

```xml
<!-- Maven: use range to always get latest 26.1.2 build -->
<version>[26.1.2.build,)</version>
<!-- Or specific build -->
<version>26.1.2.build.72-stable</version>
```

---

## Gradle Kotlin DSL

```kotlin
plugins {
    java
    id("com.github.johnrengelman.shadow") version "8.1.1"
}

group = "com.yourname"
version = "1.0.0-SNAPSHOT"

repositories {
    maven("https://repo.papermc.io/repository/maven-public/")
}

dependencies {
    compileOnly("io.papermc.paper:paper-api:26.1.2.build.72-stable")
    implementation("com.github.ben-manes.caffeine:caffeine:3.1.8")
}

java {
    toolchain.languageVersion.set(JavaLanguageVersion.of(25))
}

tasks.withType<JavaCompile> {
    options.compilerArgs.add("-parameters")
}
```

---

## Project Structure

```
your-plugin/
├── pom.xml                          # Maven config
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
# 1. Download PaperMC 26.1.2 from https://papermc.io/downloads
# 2. Create start script

# run.sh (Linux/Mac)
#!/bin/bash
java -Xms4G -Xmx4G -jar paper-26.1.2.jar nogui

# run.bat (Windows)
@echo off
java -Xms4G -Xmx4G -jar paper-26.1.2.jar nogui
pause

# 3. First run generates eula.txt, set eula=true
# 4. Place plugin jar in plugins/ folder
# 5. Start server
```

---

## Dependencies Guide

| Dependency | Purpose | Scope |
|-----------|---------|-------|
| `paper-api` | Core API | `provided` |
| `caffeine` | High-performance cache | `compile` (shade) |
| `mysql-connector-j` | MySQL database | `compile` (shade) |
| `HikariCP` | Connection pooling | `compile` (shade) |
| `sqlite-jdbc` | SQLite (rarely needed, usually built-in) | `compile` (shade) |

Always shade (relocate) third-party libraries to avoid version conflicts with other plugins.
