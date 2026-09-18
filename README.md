# PaperMC-Dev-Skills

<div align="center">

[![Minecraft](https://img.shields.io/badge/Minecraft-26.x-62B47A?logo=minecraft)](https://www.minecraft.net/)
[![Paper](https://img.shields.io/badge/Paper-26.2%20stable-2196F3?logo=papermc)](https://papermc.io/)
[![Folia](https://img.shields.io/badge/Folia-26.2%20beta-FF6D00)](https://papermc.io/software/folia)
[![Purpur](https://img.shields.io/badge/Purpur-26.2%20stable-8E44AD)](https://purpurmc.org/)
[![Java](https://img.shields.io/badge/Java-25%20LTS-007396?logo=openjdk)](https://openjdk.org/)
[![Maven](https://img.shields.io/badge/Maven-3.9-C71A36?logo=apache-maven)](https://maven.apache.org/)
[![Gradle](https://img.shields.io/badge/Gradle-8.x-02303A?logo=gradle)](https://gradle.org/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**English** | [中文](README_zh.md)

</div>

A production-grade skill package that teaches AI coding agents how to develop, debug, and maintain Minecraft server plugins for **Paper** and its two major forks, **Folia** and **Purpur**, targeting the 26.x version line and Java 25. It encodes verified version facts, build coordinates, API patterns, fork-specific threading rules, and the breaking changes introduced across 26.1, 26.2 and 26.3, so an agent can produce correct plugin code without re-deriving the ecosystem's rapid version churn.

The content is authored from primary sources: PaperMC official documentation and news posts, versioned Javadocs, the Paper/Folia/Purpur source trees, live download and Maven metadata APIs, and developer-community issue reports.

[Features](#features) | [Tech Stack](#tech-stack) | [Project Structure](#project-structure) | [Getting Started](#getting-started) | [Development](#development) | [Build & Deployment](#build--deployment) | [Skill Reference](#skill-reference) | [Version Matrix](#version-matrix) | [Troubleshooting](#troubleshooting) | [Contributing](#contributing) | [License](#license)

**Repository**: [https://github.com/IYeaSakura/PaperMC-Dev-Skills](https://github.com/IYeaSakura/PaperMC-Dev-Skills)

---

## Features

### Paper 26.x Coverage

- Documents the post-26.1 artifact versioning scheme `<minecraft-version>.build.<number>-<channel>` and the official meaning of the `alpha`, `beta` and `stable` channels
- Pins the current stable target (`paper-api:26.2.build.124-stable`) and flags the alpha-only 26.3 line
- Explains `api-version` parsing and validation rules with the exact upstream source references and the accepted error messages
- Tracks the 26.1 and 26.2 breaking changes: world storage layout, world keys, per-world clocks, beds losing their block entity, Adventure 5, and cube mob restructuring
- Replaces the deprecated Timings profiler guidance with spark

### Folia Regionised Multithreading

- Describes the regionizer model: independent regions, parallel tick loops, per-region tick counters, and the global region
- Requires and explains the `folia-supported: true` opt-in flag, including the fact that Folia refuses to load plugins without it
- Documents all four schedulers, when to use each, and why entity work must not use the region scheduler
- Covers thread-ownership checks (`isOwnedByCurrentRegion`, `isGlobalTickThread`) with a safe read-or-schedule helper
- Lists the API that is broken on Folia, with `Entity#teleport` explicitly marked as permanently unsupported in favour of `teleportAsync`
- Provides server sizing guidance, including the recommended thread allocation ceiling

### Purpur Fork Support

- Explains the drop-in relationship with Paper and confirms that Purpur-only behaviour is off by default
- Documents the complete `org.purpurmc.purpur` API surface: events, entity, language, and permission utilities
- Covers platform detection using the brand id defined by Purpur's Rebrand patch
- Inventories `purpur.yml` global and world settings that can invalidate a plugin's vanilla assumptions, such as rideable mobs, modified block behaviour, and attribute overrides
- Demonstrates an optional-integration pattern that keeps a single JAR loadable on Paper

### Cross-Fork Engineering

- Provides a single-JAR strategy that runs on Paper, Purpur, and Folia from one codebase
- Documents `teleportAsync` and scheduler usage as the portability boundary between the three servers
- Supplies a capability matrix showing exactly which APIs cost Folia compatibility
- Includes isolated-hook patterns to avoid `NoClassDefFoundError` when optional fork API is absent

### Project Setup and Persistence

- Delivers complete, copy-ready Maven and Gradle configurations with Java 25 toolchain and `release` settings
- Documents the mapping namespace manifest entry and the paperweight-userdev dev bundle workflow
- Covers SQLite, MySQL with HikariCP, and Caffeine caching with thread-safe patterns
- Includes performance and security checklists for production plugins

---

## Tech Stack

### Target Platforms

| Category | Technology | Version |
|----------|------------|---------|
| Base Server | Paper | 26.2 (build 124, stable) |
| Fork (Threading) | Folia | 26.2 (build 7, beta) |
| Fork (Features) | Purpur | 26.2 (build 2633, stable) |
| Game Version | Minecraft Java Edition | 26.2 |

### Development Toolchain

| Category | Technology | Version |
|----------|------------|---------|
| Language | Java | 25 LTS |
| Build Tool | Maven | 3.9.x |
| Build Tool | Gradle | 8.x (Kotlin DSL) |
| Core API | Bukkit / Paper API | 26.x |
| Text Library | Adventure | 5.x |
| NMS Tooling | paperweight-userdev | 2.0.0-beta.23 |

### Optional Dependencies

| Category | Artifact | Purpose |
|----------|----------|---------|
| Cache | `com.github.ben-manes.caffeine:caffeine` | High-performance in-memory cache |
| Database | `com.mysql:mysql-connector-j` | MySQL driver |
| Pooling | `com.zaxxer:HikariCP` | JDBC connection pool |
| Database | `org.xerial:sqlite-jdbc` | SQLite driver (usually provided by the server) |

### Authoring Format

| Category | Technology | Purpose |
|----------|------------|---------|
| Skill Format | Markdown with YAML front matter | Agent-discoverable skill definition |
| Skill Standard | DSH progressive-disclosure skill | Loads `SKILL.md`, expands `references/` on demand |
| Version Control | Git | Repository history |

---

## Project Structure

```
PaperMC-Dev-Skills/
├── SKILL.md                     # Skill entry point: version facts, flavor matrix, quick start
├── references/                  # Progressive-disclosure detail, loaded only when relevant
│   ├── project-setup.md         # Maven/Gradle templates, Java 25 toolchain, fork targets
│   ├── plugin-yml.md            # plugin.yml and paper-plugin.yml specification
│   ├── api-patterns.md          # Events, commands, schedulers, GUI, items, multi-target guide
│   ├── data-storage.md          # SQLite, MySQL/HikariCP, Caffeine, async patterns
│   ├── version-matrix.md        # Version matrix, api-version rules, migration, NMS
│   ├── folia.md                 # Regionised threading, schedulers, Folia broken-API list
│   └── purpur.md                # Purpur fork API, purpur.yml options, safe integration
├── README.md                    # This file (English)
├── README_zh.md                 # Chinese documentation
└── LICENSE                      # MIT license
```

### File Responsibilities

| File | Role | Approximate Size |
|------|------|------------------|
| `SKILL.md` | Always-loaded summary: facts, flavor selection, quick start, checklists | ~400 lines |
| `references/version-matrix.md` | Canonical version and compatibility data, including the Paper Family comparison | ~640 lines |
| `references/api-patterns.md` | Runnable API examples plus the multi-target engineering section | ~950 lines |
| `references/folia.md` | Complete Folia guide with the broken-API table and migration checklist | ~230 lines |
| `references/purpur.md` | Complete Purpur guide with the fork API inventory and config coverage | ~270 lines |
| `references/project-setup.md` | Build configuration for all three targets | ~430 lines |
| `references/plugin-yml.md` | Metadata file specification, including `folia-supported` | ~330 lines |
| `references/data-storage.md` | Persistence and caching patterns | ~380 lines |

---

## Getting Started

### Prerequisites

- **DSH-compatible agent runtime** with the skills directory available to the session
- **Git** 2.30 or higher to clone the repository
- **No build toolchain is required to use the skill.** Java, Maven, and Gradle are only needed when you act on the skill's output to build an actual plugin
- To build and test generated plugins: **Java 25 LTS** and either **Maven 3.9+** or **Gradle 8+**

### Installation

Install the skill into a session skills directory. The directory name determines the skill's on-disk location; the skill's identity comes from the `name` field in its front matter.

```bash
# Clone the repository
git clone https://github.com/IYeaSakura/PaperMC-Dev-Skills.git

# Install into the session skills directory (Windows default path shown)
# C:\Users\<user>\.dsh\skills\
mv PaperMC-Dev-Skills "%USERPROFILE%\.dsh\skills\Minecraft-Paper-Dev-Skills"
```

```bash
# Linux / macOS equivalent
git clone https://github.com/IYeaSakura/PaperMC-Dev-Skills.git
mv PaperMC-Dev-Skills ~/.dsh/skills/Minecraft-Paper-Dev-Skills
```

### Verifying the Installation

```bash
# Confirm the expected layout is present
ls -1 ~/.dsh/skills/Minecraft-Paper-Dev-Skills
# Expected: LICENSE  README.md  README_zh.md  SKILL.md  references
```

Once installed, the skill appears in the session skill catalog as `minecraft-paper-dev-skills`. Invoke it explicitly, or let the runtime select it when a request matches its description.

### Configuration

The skill is documentation and requires no configuration files or environment variables. Two optional adjustments are useful:

| Setting | Purpose | Notes |
|---------|---------|-------|
| Directory name | Placement inside the skills root | Verify against the runtime's skill discovery convention |
| Target versions | Change the pinned Paper/Folia/Purpur builds | Edit the version facts in `SKILL.md` and `references/version-matrix.md` |

### Version Snapshot

The skill embeds a verified snapshot dated **2026-09-17**. Re-check the live endpoints before relying on it for a new project:

```bash
# Paper versions and channel status
curl -s https://fill.papermc.io/v3/projects/paper/versions/26.2/builds | head -c 400

# Folia versions
curl -s https://fill.papermc.io/v3/projects/folia/versions/26.2/builds | head -c 400

# Purpur versions
curl -s https://api.purpurmc.org/v2/purpur/
```

---

## Development

### Editing the Skill

All content is Markdown. Follow these conventions to keep the package coherent:

1. Keep `SKILL.md` a summary. Move exhaustive tables and long examples into `references/` and link them from `SKILL.md` and from the reference table at the bottom of the file.
2. Update the version snapshot date in `SKILL.md` whenever version facts change.
3. Cross-link between references so a reader can navigate from a summary to the detail and back.
4. Cite primary sources for any newly added factual claim.

### Content Conventions

| Item | Convention | Example |
|------|------------|---------|
| Headings | Sentence case, imperative where instructional | `## Marking a Plugin as Folia-Supported` |
| Code fences | Always tagged with a language | `java`, `kotlin`, `yaml`, `bash` |
| Version strings | Backticked, exact artifact form | `26.2.build.124-stable` |
| Class references | Fully qualified on first use | `io.papermc.paper.threadedregions.scheduler.RegionScheduler` |
| Platform names | Capitalized as branded | Paper, Folia, Purpur |
| Deprecation claims | State the replacement explicitly | `Use teleportAsync` |
| Tables | Used for matrices and comparisons | Version, capability, and config tables |

### Validation Workflow

Because the skill encodes fast-moving version facts, validate changes against live sources:

```bash
# 1. Confirm the Paper API artifact still resolves
curl -s https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/maven-metadata.xml \
  | grep -o '<version>26\.[0-9.]*build\.[0-9]*-stable</version>' | tail -n 3

# 2. Confirm the Purpur API artifact still resolves
curl -s https://repo.purpurmc.org/snapshots/org/purpurmc/purpur/purpur-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3

# 3. Confirm the Folia API artifact still resolves
curl -s https://repo.papermc.io/repository/maven-public/dev/folia/folia-api/maven-metadata.xml \
  | grep -o '<version>[^<]*</version>' | tail -n 3
```

### Review Checklist

Before committing a content change:

- [ ] Any changed version number is verified against a live endpoint or versioned Javadoc
- [ ] New API claims name the class and method, not a paraphrase
- [ ] Breaking changes state whether they are source breaks, binary breaks, or deprecations
- [ ] Fork-specific claims say which server they apply to
- [ ] `SKILL.md` remains a summary and does not duplicate whole reference sections
- [ ] The version snapshot date reflects the latest verification

---

## Build & Deployment

This project has no compilation step; "build" means packaging and "deployment" means distribution to consumers of the skill.

### Producing a Distribution

```bash
# Create a release archive containing only tracked files
git archive --format=zip --output=PaperMC-Dev-Skills.zip HEAD

# Or produce a tarball
git archive --format=tar.gz --output=PaperMC-Dev-Skills.tar.gz HEAD
```

### Deployment Targets

| Target | Method | Notes |
|--------|--------|-------|
| Local agent session | Copy into the session skills directory | Primary usage |
| Team distribution | Commit to an internal mirror | The `references/` directory must be included |
| Public distribution | Publish to GitHub and tag releases | Consumers clone into their skills root |
| Offline environments | Ship the release archive | No network access is required to read the skill |

### Release Procedure

```bash
# Tag a release after content updates
git tag -a v26.2.0 -m "Skill content updated for Paper 26.2 / Folia 26.2 / Purpur 26.2"
git push origin v26.2.0
```

### Suggested Versioning

| Component | Meaning | Example |
|-----------|---------|---------|
| Major | Reserved for a restructuring of the skill layout | `1.0.0` |
| Minor | New platform coverage or a new reference file | `0.2.0` |
| Patch | Factual corrections and version bumps | `0.2.1` |

---

## Skill Reference

### Skill Metadata

| Field | Value |
|-------|-------|
| Skill name | `minecraft-paper-dev-skills` |
| Entry point | `SKILL.md` |
| Invocation | Explicitly by name, or by description match |
| Loading model | Progressive: `SKILL.md` first, `references/` on demand |

### Reference File Index

| Topic | File | Contents |
|-------|------|----------|
| Paper 26.x overview and quick start | `SKILL.md` | Version facts, flavor matrix, project setup, checklists |
| Build configuration | `references/project-setup.md` | Maven and Gradle templates, Java 25 toolchain, fork dependencies |
| Plugin metadata | `references/plugin-yml.md` | `plugin.yml` and `paper-plugin.yml`, `api-version`, `folia-supported` |
| API patterns | `references/api-patterns.md` | Events, commands, schedulers, GUI, items, entities, multi-target guide |
| Data persistence | `references/data-storage.md` | SQLite, MySQL, HikariCP, Caffeine, async bridges |
| Versions and migration | `references/version-matrix.md` | Version matrix, Paper Family comparison, `api-version` rules, NMS |
| Folia | `references/folia.md` | Regionised threading, schedulers, thread ownership, broken API |
| Purpur | `references/purpur.md` | Fork API, `purpur.yml`, permissions, safe integration |

### Version Detection Reference

| Server | Latest Version Endpoint | Artifact Coordinate |
|--------|-------------------------|---------------------|
| Paper | `https://fill.papermc.io/v3/projects/paper` | `io.papermc.paper:paper-api` |
| Folia | `https://fill.papermc.io/v3/projects/folia` | `dev.folia:folia-api` |
| Purpur | `https://api.purpurmc.org/v2/purpur/` | `org.purpurmc.purpur:purpur-api` |

---

## Version Matrix

As of the **2026-09-17** snapshot:

| PaperMC | Minecraft | Min Java | api-version | Status |
|---------|-----------|----------|-------------|--------|
| 1.20.5 - 1.20.6 | 1.20.5 - 1.20.6 | 21 | `1.20` | Legacy |
| 1.21 - 1.21.11 | 1.21 - 1.21.11 | 21 | `1.21` | Legacy |
| 26.1.1 | 26.1.1 | 25 | `26.1` | Unsupported |
| 26.1.2 | 26.1.2 | 25 | `26.1` or `26.1.2` | Supported |
| **26.2** | **26.2** | **25** | **`26.2`** | **Latest stable** |
| 26.3 | 26.3 | 25 | `26.3` | Paper alpha only |

### Fork Comparison

| | Paper | Folia | Purpur |
|---|---|---|---|
| Relationship | Base server | Paper fork with regionised multithreading | Paper drop-in replacement |
| Threading | One main thread | No main thread; one tick loop per region | One main thread |
| Latest 26.2 build | `124-stable` | `7-beta` | `2633-stable` |
| Maven group | `io.papermc.paper` | `dev.folia` | `org.purpurmc.purpur` |
| Pre-release channel name | `alpha` | `beta` | `experimental` |
| Plugin opt-in required | No | `folia-supported: true` | No |
| Ordinary Paper plugins load | Yes | Almost none | Yes, unchanged |

---

## Troubleshooting

### Skill Not Discovered

**Problem**: The skill does not appear in the session skill catalog.

**Solution**:
- Confirm the skill directory sits directly under the skills root, not nested one level deeper
- Confirm `SKILL.md` exists at the skill directory root and its front matter contains both `name` and `description`
- Confirm the directory name convention matches what your runtime expects for skill discovery

### Reference Files Not Followed

**Problem**: The agent summarizes from `SKILL.md` but never opens `references/`.

**Solution**:
- Check that relative links in `SKILL.md` resolve, for example `references/folia.md`
- Ask explicitly for the topic, such as "read the Folia reference and apply it"
- Keep `SKILL.md` short; oversized entry points discourage reference expansion

### Outdated Version Facts

**Problem**: The skill recommends a build that is no longer the newest stable release.

**Solution**:
- Re-query the endpoints in [Version Snapshot](#version-snapshot)
- Update the version facts in `SKILL.md` and the matrix in `references/version-matrix.md`
- Update the snapshot date and commit the change

### Wrong Fork Targeted

**Problem**: Advice for Folia is applied to a Paper or Purpur server, or the reverse.

**Solution**:
- Check the fork comparison table above before applying scheduler or threading advice
- Treat Folia's constraints as additive: API that is broken there is still valid on Paper and Purpur
- Remember that Purpur inherits Paper's single-threaded model and is not a Folia fork

### api-version Rejected by the Server

**Problem**: The server logs `Unsupported API version` or refuses to load the plugin.

**Solution**:
- Use `major.minor` such as `26.2`, not a build id such as `26.2.build.124-stable`
- Never use a `1.26.x` form; the 2026 scheme is `year.drop`
- Check `settings.minimum-api` in `bukkit.yml` if the plugin is rejected for being too old

---

## Contributing

Contributions are welcome. Accuracy of version facts is the primary quality bar.

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/your-topic`
3. Make changes following the content conventions above
4. Verify every version number against a live endpoint or versioned Javadoc
5. Commit: `git commit -m 'docs: update Paper 26.x version facts'`
6. Push: `git push origin feature/your-topic`
7. Open a Pull Request

### Code Quality Requirements

Before submitting a PR:

- [ ] Every version number is verified against an authoritative source
- [ ] Every API claim names a real class and method
- [ ] Fork-specific guidance is labelled with the server it applies to
- [ ] Breaking changes are classified as source break, binary break, or deprecation
- [ ] `SKILL.md` links resolve to existing files
- [ ] The snapshot date is updated if version facts changed
- [ ] No emoji in body text

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) file for details.

```
MIT License

Copyright (c) 2026 Yuyang.Wang

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## Acknowledgments

This skill is assembled from the documentation and source of the projects it covers:

- [PaperMC](https://papermc.io/) - Paper server, development documentation, and versioned Javadocs
- [Folia](https://github.com/PaperMC/Folia) - Regionised multithreading model and its published broken-API list
- [PurpurMC](https://purpurmc.org/) - Purpur fork, configuration documentation, and Javadocs
- [Adventure](https://docs.advntr.dev/) - Component-based text API shipped with Paper 26.2
- [paperweight](https://github.com/PaperMC/paperweight) - Gradle tooling for server-internals development
- [Mojang Studios](https://www.minecraft.net/) - Minecraft Java Edition

---

## Contact

- **Author**: Yuyang.Wang
- **Website**: [https://sakurain.net](https://sakurain.net)
- **Email**: [Yae_SakuRain@outlook.com](mailto:Yae_SakuRain@outlook.com)
- **GitHub**: [https://github.com/IYeaSakura](https://github.com/IYeaSakura)

---

<p align="center">
  Made by Yuyang.Wang
</p>
