# The Plugin Project Architecture (Hub-and-Spoke)

This document outlines the standardized architectural pattern for building "Plugin Projects" in `socialscrape` / `gscrape`.
This pattern ensures that capabilities are modular, reusable (via direct Python imports), and independently accessible via AI agents, while maintaining a unified interface (TUI/CLI) at the project root.

> **PPA Framework**: **ROOT** — this is the entry point of the framework tree. Every other PPA document specializes one aspect of the Hub-and-Spoke concept defined here.
> **Framework Map & Maintenance Protocol**: [agent.md](agent.md)
> **Child documents (Hub)**: [CLI/TUI Hub Architecture](cli_architecture.md) · [Agentic CLI Architecture](agentic_cli_architecture.md) · [Bulk Scraper Engine Specification](bulk_scraper_engine_specification.md) · [Deployment Architecture](deployment_architecture.md)
> **Child documents (Spokes)**: [Plugin SDK Specification](plugin_sdk_specification.md)

## Framework Document Map

```text
plugin_project_architecture.md  ← ROOT (Hub-and-Spoke concept)
│
├── HUB (root-level interface: TUI, agent, storage, packaging)
│   ├── cli_architecture.md                 Universal CLI/TUI Hub standard (visual, autocomplete, packaging)
│   │   └── agentic_cli_architecture.md     Agentic extension (AI mode, sessions, OpenRouter tool calling)
│   ├── bulk_scraper_engine_specification.md  Batch & resilience engine (batch_manager.py, 15 patterns)
│   └── deployment_architecture.md          Packaging, dual-mode distribution & storage resolution
│
└── SPOKES (plugins/<name>/)
    └── plugin_sdk_specification.md         Plugin SDK standard (core.py, cli.py, SKILL.md, Outputs/)
```

Cross-cutting links (non-tree): `agentic_cli_architecture.md` ↔ `bulk_scraper_engine_specification.md` (tool dispatch & `/resume-batch`) · `agentic_cli_architecture.md` ↔ `plugin_sdk_specification.md` (tool schemas ↔ `SKILL.md`) · `bulk_scraper_engine_specification.md` ↔ `plugin_sdk_specification.md` (decoupled `core.py` telemetry) · `deployment_architecture.md` ↔ `cli_architecture.md` (packaging) · `deployment_architecture.md` ↔ `plugin_sdk_specification.md` (plugin layout & `Outputs/`).

## 1. The Core Concept: Hub-and-Spoke

The architecture is divided into two distinct layers:

1. **The Hub (Root Level Interface):** The main project root which houses the Universal TUI (Text User Interface), MCP Server, or Streamlit app. This layer contains **no scraping logic** itself. It merely imports and orchestrates the plugins.
2. **The Spokes (Plugins):** Self-contained modular folders inside the `plugins/` directory. Each plugin defines a specific capability (e.g., `youtube-scrape`, `tiktok-scrape`, `gmaps_scraper`) and provides both the executable logic and the AI agent instructions.

```text
socialscrape/                   <-- THE HUB (Decoupled Layered Orchestration)
├── ui.py                       # Presentation Layer (Rich themes, OpenCode banners, cards, toolbar)
├── agent.py                    # AI Core (OpenRouter model whitelist, multi-turn memory, stateless schemas)
├── handlers.py                 # Command & Subprocess Layer (Sandboxed runners, modal drilldowns)
├── batch_manager.py            # Batch & Resilience Engine (State persistence, retries, jitter, recovery)
├── storage.py                  # Storage & Path Engine (Dual-Mode Global ~/.<app> vs Local Dev)
├── socialscrape_cli.py         # OpenCode Master TUI Controller (Terminal lifecycle, autocomplete, REPL)
├── cli.py                      # Local CLI runner shortcut (python cli.py)
├── pyproject.toml              # Global CLI packaging (socialscrape & socialscrape-dev scripts)
├── requirements.txt            # Root aggregated dependencies
├── README.md                   # Master documentation
├── SKILL.md                    # Master Hub Agent Skill (Pattern 4 Orchestrator Manifest)
├── batches/                    # Fail-safe batch state checkpoints (.json)
├── chats/                      # Multi-turn conversation session logs (.json)
│
└── plugins/                    <-- THE SPOKES (Isolated Capabilities)
    ├── shared/                 # Shared session manager & rate limiter
    ├── youtube-scrape/         # Individual Plugin A
    ├── instagram-scrape/       # Individual Plugin B
    ├── tiktok-scrape/          # Individual Plugin C
    ├── linkedin-scrape/        # Individual Plugin D
    └── twitter-scrape/         # Individual Plugin E
```

## 2. Anatomy of a Plugin SDK

Every plugin inside the `plugins/` folder MUST follow **Pattern 4 (Full Suites / Plugin Bundle)** as defined in the global `skill-creator` structural patterns (`references/structural-patterns.md`).

While Pattern 4 allows for flexible "Hybrid Composition" (including `references/`, `examples/`, etc.), any plugin built within this project that relies on Python scripts **MUST** implement the **Plugin SDK** standard:

1. **`core.py` (The Plugin SDK):** The pure logic that performs the actual work (scraping, downloading). It MUST return structured data and MUST NOT contain `print()` or `sys.exit()` calls.
2. **`cli.py` & Utility Scripts:** Thin wrappers around `core.py` that handle `argparse` and printing to the console for standalone terminal usage.
3. **`Outputs/` Folder:** All generated data must remain isolated inside this standardized folder. If the plugin has different types of outputs, they MUST be categorized into subfolders (e.g., `Outputs/videos/`, `Outputs/live/`, `Outputs/shorts/`).

For the full detailed breakdown of `plugin.json`, `SKILL.md`, `requirements.txt`, and Hybrid Compositions, refer to the global skill documentation in `structural-patterns.md`.

### 1. `core.py` (The Plugin SDK)

This is the heart of the plugin. It contains **pure Python functions** that perform the actual work (scraping, downloading).

- **Rule:** It MUST return structured data (dictionaries, lists, or Dataclasses).
- **Rule:** It MUST NOT contain `print()`, `input()`, `sys.argv`, or `sys.exit()` calls.
- **Why?** So that the Root TUI (`main_tui.py`) or another plugin can safely do `from plugins.youtube_scrape.core import get_transcript` in-memory without polluting the terminal or crashing the program.

### 2. `cli.py` (The Local Interface)

A thin wrapper around `core.py`.

- **Purpose:** Allows users (or agents) to run this specific plugin standalone from the terminal (e.g., `python plugins/youtube-scrape/cli.py --url XYZ`).
- **Rule:** This file handles all the `argparse`, `print()`, and interactive wizard prompts. It imports from `core.py` to get the data, then prints it to the console.

### 3. Utility CLI Scripts (`run_batch.py`)

Plugins can have additional CLI scripts for specific workflows (like `run_batch.py` for processing lists of URLs). These scripts should also import from `core.py` and act as specialized command-line tools for the plugin.

### 4. `SKILL.md` (Agent Manifest)

The markdown file that explains to Antigravity (or other agents) what this plugin does, how to call the CLI, and what parameters it accepts. It transforms the raw code into an "Agentic Skill".

### 5. `plugin.json`

Metadata regarding the plugin name, version, description, and entry points.

### 6. `README.md`

Documentation specific to the plugin. Explains how to use the plugin standalone and what data it outputs. The Root `README.md` will link to this file.

### 7. `requirements.txt`

Dependencies strictly required for this plugin to run (e.g., `yt-dlp`, `requests`).

### 8. `Outputs/`

A strictly named folder where the plugin stores its local artifacts. Standardizing this prevents clutter across the repository. Folders should use snake_case (e.g., `live_outputs`, not `live outputs`).

### 9. `run_logs/` (User Parameter Logging)

An isolated folder where the plugin logs operator run requests and parameter configurations (`runs.csv`).
- **Rule:** Configurable and scraping plugins must log the exact user inputs (selected catalog, industries, countries, sampled states, location counts, max limits, modes) before initiating execution.
- **Why?** Preserves complete audit history, enables operator inspection, and allows intelligent state comparison to prevent redundant re-scraping of previously completed targets.

## 3. Root Level Linking (Dependencies & Docs)

To keep the project clean, the Root Hub links down to the Plugins instead of duplicating information:

- **Dependencies:** The root `requirements.txt` aggregates the dependencies of all plugins. It can do this using standard pip referencing: `-r plugins/youtube-scrape/requirements.txt`.
- **Documentation:** The root `README.md` acts as a directory. Instead of explaining how YouTube Scraping works, it simply provides a link to `plugins/youtube-scrape/README.md`.
- **Agent Skill Orchestration:** The root `SKILL.md` serves as the master agent-native index. It indexes all child spoke skills (`plugins/<spoke>/SKILL.md`), defines the universal command matrix, and details in-memory `--stdout` execution so external coding agents (Codex, Claude, Antigravity) can navigate and execute tools with zero friction.


## 4. How Data Flows

1. **User/Agent Invocation:** An agent or user runs a command via the Root Hub (e.g. `gscrape` or `socialscrape`).
2. **Subprocess/In-Memory Dispatch:** The Hub delegates execution to the appropriate spoke plugin (e.g., `plugins/gmaps_scraper/cli.py` or directly calling `core.py`).
3. **Execution & Event Emission:** `core.py` executes scraping with full stealth & batch resilience, emitting live telemetry events (`info`, `progress`, `warning`, `error`, `success`).
4. **Data Persistence:** Outputs are saved atomically to `Outputs/` or `~/.<tool>/outputs/` and recorded in `run_manifest.csv` and `master.csv`.

---

## 5. The Canonical Schema Whitelist Law (Immutable Output Schemas)

In a multi-plugin hub-and-spoke ecosystem, pipeline spokes continuously enrich shared datasets. Without strict schema locking, downstream plugins or metadata expanders dynamically inject ad-hoc columns, shifting column offsets, corrupting CSV tabular alignment, and creating unindexed "spillover" attributes.

The **Canonical Schema Whitelist Law** establishes strict, immutable data boundary invariants:

1. **Master Table Invariance**:
   - Master CSV ledgers (`company.csv`, `people.csv`) possess strict, pre-defined canonical schemas locked at the storage layer (e.g., `CANONICAL_COMPANY_FIELDS`, `CANONICAL_PEOPLE_FIELDS`).
   - No spoke plugin, enrichment step, or third-party tool is permitted to append arbitrary keys to master tables.
2. **Strict Whitelist Filtering on Write**:
   - Storage write operations (`upsert_company`, `update_company_in_place`, `upsert_person`, `update_person_in_place`) MUST strictly whitelist records against canonical fields (`{k: v for k, v in record.items() if k in target_fields}`) prior to disk serialization.
   - Dynamic key expansion loops (`for k in record.keys(): file_fields.append(k)`) are **strictly prohibited**. Any unwhitelisted keys present in memory payloads are discarded at write time.
3. **The Single-Column Social Link Standard**:
   - All social media URLs and platform identities MUST be coalesced into a single, unified, comma-separated `social_links` column (e.g., `linkedin: https://..., twitter: https://...`).
   - Disparate platform columns (`linkedin_url`, `twitter_url`, `instagram_url`, etc.) are prohibited in master company and people ledgers.
4. **Lossless Atomic Header Upgrades**:
   - If an existing output CSV on disk is missing canonical columns (e.g., legacy files created before pipeline expansion), storage engines MUST upgrade the CSV header atomically to the full canonical schema on write, preserving all existing record values without column displacement.

