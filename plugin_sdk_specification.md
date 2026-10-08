# SocialScrape Plugin SDK & Architectural Specification 📐🚀

> **Document Type**: Master Architectural Specification & Design Standard  
> **Version**: 2.0.0  
> **Target Audience**: Core Developers, Plugin Authors, and AI Coding Agents (Claude, Codex, Antigravity)  
> **Location**: `.framework/PPA/plugin_sdk_specification.md`

> **PPA Framework**: Spoke layer — the standard every plugin (spoke) must implement.
> **Upstream (Root)**: [Plugin Project Architecture](plugin_project_architecture.md)
> **Related**: [Agentic CLI Architecture](agentic_cli_architecture.md) — Hub tool schemas are generated from each plugin's `SKILL.md` · [Bulk Scraper Engine Specification](bulk_scraper_engine_specification.md) — Pattern 5: `core.py` publishes telemetry, never renders · [Deployment Architecture](deployment_architecture.md) — where `Outputs/` resolves in global vs local mode · [agent.md](agent.md) — framework map

---

## 1. Executive Summary & Hub-and-Spoke Topology

**SocialScrape** is built on a modular **Hub-and-Spoke Plugin Architecture**. The project is strictly bifurcated into two distinct operational layers:

1. **The Hub (Root Orchestrator - `socialscrape_cli.py`)**:
   - Manages the interactive OpenCode-style terminal user interface (TUI), stateful session persistence (`chats/`), command palette, persistent autocomplete, and natural language AI tool routing via OpenRouter.
   - **Zero Scraping Logic**: The Hub contains zero scraping, parsing, or downloading code. It discovers, inspects, and executes plugins via sandboxed child processes.
2. **The Spokes (Self-Contained Plugin SDKs - `plugins/<plugin-name>/`)**:
   - Each plugin is a fully independent, isolated software package providing pure Python SDK functions (`core.py`), a local interactive CLI runner (`cli.py`), documentation (`README.md`), agent parameter schemas (`SKILL.md`), metadata manifests (`plugin.json`), and categorized data storage (`Outputs/`).

```text
                               ┌────────────────────────────────────────────────────────┐
                               │       THE HUB: Master Orchestrator & Presentation      │
                               │               (`socialscrape_cli.py`)                  │
                               └───────────────────────────┬────────────────────────────┘
                                                           │
                                ┌──────────────────────────┼──────────────────────────┐
                                ▼                          ▼                          ▼
                   ┌────────────────────────┐ ┌────────────────────────┐ ┌────────────────────────┐
                   │ plugins/youtube-scrape │ │plugins/instagram-scrape│ │ plugins/tiktok-scrape  │
                   └────────────────────────┘ └────────────────────────┘ └────────────────────────┘
```

---

## 2. Standard Plugin File Layout & Anatomy

Every plugin inside `plugins/<plugin-name>/` MUST implement the following directory and file layout:

```text
plugins/<plugin-name>/
├── core.py               # ⚡ Pure Python SDK: Logic layer (zero prints, returns dicts/dataclasses)
├── cli.py                # 🧙 Presentation Layer: Interactive wizard & argparse CLI runner
├── plugin.json           # 📋 Universal Plugin Manifest: Metadata, version, description, entrypoint
├── SKILL.md              # 🧠 AI Agent Protocol: Parameter definitions & tool invocation rules
├── README.md             # 📖 Master Documentation: 7-section standard documentation
├── requirements.txt      # 📦 Isolated Dependencies: Pinned libraries required for this plugin
├── .gitignore            # 🔒 Local Git Exclusions: Ignores downloaded media and caches
├── run_logs/             # 📜 Historical Operational Audit Logs
│   └── runs.csv          # Run parameter logs (timestamp, catalog, filters, counts, mode)
└── Outputs/              # 💾 Standardized Categorized Local Storage
    ├── videos/           # Downloaded MP4 video streams, reels, and clips
    ├── thumbnails/       # High-resolution cover images, photos, and thumbnails
    ├── comments/         # Structured comment threads and reply JSONs
    └── text_posts/       # Transcripts, biographies, tweets, and Markdown profile summaries
```

---

## 3. Pattern 1: Pure Logic SDK Standard (`core.py`)

`core.py` is the algorithmic core of the plugin. It MUST be importable in-memory by any Python process without producing side effects.

### Mandatory Rules for `core.py`:
1. **Zero Console Side-Effects**: `core.py` MUST NOT call `print()`, `input()`, `sys.exit()`, or read `sys.argv`.
2. **Structured Returns**: All functions MUST return typed dictionaries, dataclasses, or lists containing predictable keys (including a `"status"` key with values `"success"` or `"error"`).
3. **Graceful Error Encapsulation**: Network exceptions or parsing failures must be caught, logged internally, and returned as `{"status": "error", "error": "Human-readable reason"}` rather than crashing the calling process.
4. **Isolated Default Outputs**: Functions that save files must accept an optional `output_dir: Optional[Union[Path, str]] = None` parameter that defaults to the plugin's local `Outputs/` directory.

### Reference Code Pattern:
```python
def get_item_data(target_id: str, output_dir: Optional[Path] = None) -> Dict[str, Any]:
    """Pure SDK function returning structured data in RAM."""
    result = {
        "status": "success",
        "id": target_id,
        "title": None,
        "content": None,
        "saved_path": None,
        "error": None
    }
    try:
        # Perform extraction logic
        ...
        return result
    except Exception as exc:
        result["status"] = "error"
        result["error"] = str(exc)
        return result
```

---

## 4. Pattern 2: Interactive Presenter & CLI Standard (`cli.py`)

`cli.py` is the presentation and execution wrapper around `core.py`.

### Mandatory Rules for `cli.py`:
1. **Argparse CLI Support**: Accepts direct positional arguments and flags (e.g. `--download-video`, `--comments`, `--profile`, `--all`, `--stdout`, `-q/--quality`).
2. **Interactive Wizard (`-i / --interactive`)**: When launched without arguments or with `-i`, boots a step-by-step guided prompt allowing manual operators to choose extraction actions and quality levels.
3. **In-Memory Pipe (`--stdout`)**: Supports dumping raw JSON or plain text directly to standard output for AI agent consumption.
4. **UTF-8 Output Reconfiguration**: Enforces `sys.stdout.reconfigure(encoding="utf-8")` at the top of the file to prevent Windows terminal Unicode encoding crashes.
5. **Batch Progress Feedback Standard**: Any CLI handler executing batch or bulk operations (e.g. multi-location scraping, CSV enrichment, batch verification, batch auditing) MUST render visual progress via `rich.progress.Progress` with standard columns (`SpinnerColumn`, `TextColumn`, `BarColumn`, `TaskProgressColumn`, `MofNCompleteColumn`, `TimeElapsedColumn`), subscribing to the generator or event callback emitted by `core.py`.
6. **Run Parameter Audit Standard**: Every execution invoked via interactive wizard or CLI arguments that accepts operational parameters (e.g., catalogs, industry filters, countries, target states, generated counts, limits, modes) MUST log the execution configuration into `run_logs/runs.csv` via the shared `OutputManager.log_run_parameters()` or equivalent localized logger, ensuring full reproducibility and auditability across all runs.

---

## 5. Pattern 3: Standardized Plugin README Specification (7-Section Blueprint)

Every plugin `README.md` must follow the **7-Section Architecture Blueprint** to guarantee consistency across all platforms:

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. Title & High-Level Value Proposition (Platform badge + 1-sentence intro) │
├─────────────────────────────────────────────────────────────────────────────┤
│ 2. 🌟 5-Pillar Comprehensive Feature Breakdown                              │
│    • Under-the-hood mechanics                                               │
│    • Concrete list of data fields extracted                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ 3. 📁 Standardized Outputs/ Hierarchy (Tree diagram of subfolders & files) │
├─────────────────────────────────────────────────────────────────────────────┤
│ 4. 💡 AI Agent Usage: In-Memory / Zero-Disk Execution                       │
│    • In-memory CLI pipe (--stdout)                                          │
│    • Pure Python SDK import snippet in RAM                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│ 5. 🚀 CLI Commands & Usage Examples                                         │
│    • Interactive wizard (-i)                                                │
│    • Single URL scrapes & quality resolution flags                          │
│    • Full extraction suite (--all)                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ 6. ⚙️ CLI Flag Reference Table (Flag, Shorthand, Type, Default, Description)│
├─────────────────────────────────────────────────────────────────────────────┤
│ 7. 🔒 Security, Rate-Limiting Rails & Defensive Engineering                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Pattern 4: Shared Stealth & Session Management Protocol

Located in `plugins/shared/session_manager.py` and central `storage.py`, this shared protocol centralizes token storage, rate-limiting, and stealth mechanics for all scrapers.

```text
                               ┌─────────────────────────────────┐
                               │  plugins/shared/session_manager │
                               └────────────────┬────────────────┘
                                                │
                     ┌──────────────────────────┴──────────────────────────┐
                     ▼                                                     ▼
        ┌─────────────────────────┐                           ┌─────────────────────────┐
        │       RateLimiter       │                           │     SessionManager      │
        ├─────────────────────────┤                           ├─────────────────────────┤
        │ • Hard Daily Ceilings   │                           │ • Burner Token Storage  │
        │ • Hourly Request Caps   │                           │ • Token Masking (Logs)  │
        │ • Human Delay Jitter    │                           │ • Header Translation    │
        └─────────────────────────┘                           └─────────────────────────┘
```

### Safety Rail Specifications:
| Platform | Max Requests / Day | Max Requests / Hour | Delay Pacing Range | Managed Credential |
| :--- | :--- | :--- | :--- | :--- |
| **LinkedIn** | `50 req/day` (Hard ceiling) | `15 req/hour` | `4.0s – 8.0s` | `li_at` cookie |
| **Instagram** | `500 req/day` | `100 req/hour` | `2.0s – 5.0s` | `sessionid` cookie |
| **TikTok** | `1,000 req/day` | `150 req/hour` | `1.5s – 4.0s` | `sessionid` / Web tokens |
| **Twitter / X** | `1,500 req/day` | `250 req/hour` | `1.0s – 3.0s` | Zero-cookie / `auth_token` |

### Core Mechanics:
1. **Token Masking**: The `SessionManager.mask_token(token)` method automatically redacts credentials for safe console logging (e.g. `mock...7890`).
2. **Session Persistence**: Sessions are persisted in Git-ignored `.sessions.json` to prevent accidental credential leakage in version control.
3. **Randomized Jitter**: The `RateLimiter.check_and_pace(platform, alias)` method calculates elapsed time since the previous request and sleeps for `random.uniform(min_delay, max_delay)` to replicate natural human browsing patterns.
4. **Automated Consent & Cookie Wall Dismissal (`dismiss_consent_banners`)**:
   - Every browser-driven spoke plugin (`gmaps-scraper`, `social-deepscrape`, `social-verify`, `lead-ai-auditor`) MUST invoke automated consent banner dismissal immediately after initial page navigation and during selector wait loops.
   - Probes multi-lingual selectors (`button:has-text('Accept all')`, `button:has-text('Tout accepter')`, `button:has-text('Alle akzeptieren')`, `button:has-text('Acepto')`, `form[action*='consent'] button`, `button[aria-label*='Accept']`, `#L2AGLb`) with a non-blocking timeout (1-2s).
   - Guarantees unattended automation continues seamlessly without hanging on regional GDPR or consent modals.

---

## 7. Pattern 5: Hybrid Extraction Architecture

Scrapers decouple **Media Downloading** from **Structured Data Scraping**:

```text
                                    ┌───────────────────────┐
                                    │   Target Social URL   │
                                    └───────────┬───────────┘
                                                │
                        ┌───────────────────────┴───────────────────────┐
                        ▼                                               ▼
            ┌────────────────────────┐                      ┌────────────────────────┐
            │   Media Engine Layer   │                      │  Data Extraction Layer │
            │       (`yt-dlp`)       │                      │  (Zero-Cookie Web APIs)│
            ├────────────────────────┤                      ├────────────────────────┤
            │ • 4K / 1080p / 720p MP4│                      │ • User Profile & Bio   │
            │ • Watermark-free Clips │                      │ • Captions / Speech    │
            │ • Isolated Audio (M4A) │                      │ • Structured Comments  │
            │ • High-Res Thumbnails  │                      │ • Timeline Feeds       │
            └────────────────────────┘                      └────────────────────────┘
```

*   **Media Layer (`yt-dlp`)**: Leverages battle-tested stream extractors with format selectors (`bestvideo[height<=1080]+bestaudio/best`) to handle CDN signature deciphering.
*   **Data Layer (Lightweight APIs)**: Uses zero-cookie endpoints (Twitter Syndication API, YouTube Android Player client, Instagram Web GraphQL) for sub-second text and metadata extraction with zero headless browser overhead.

---

## 8. Pattern 6: In-Memory / Zero-Disk AI Stream Pattern

AI Agents (Claude Code, Cursor, Antigravity) running automated workflows do not need to fill user disks with multi-gigabyte media files when they only require text, transcripts, or engagement numbers.

### Requirements:
1. **`--stdout` Flag**: Every plugin CLI must support `--stdout` to write clean, parseable JSON or plain text directly to the process output stream.
2. **Pure SDK Invocation**: Agent scripts can import `plugins.<name>.core` directly to retrieve data structures in RAM in under 500 milliseconds.

---

## 9. Pattern 7: Subprocess Fault-Isolation & Dynamic Discovery

The master CLI Hub (`socialscrape_cli.py`) maintains total process isolation from individual plugins.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        Master Hub Process                              │
│                                                                        │
│  1. Scans `plugins/*/plugin.json` to discover active spokes.           │
│  2. Registers slash commands (/youtube, /instagram, /tiktok, etc.).    │
│  3. Maps OpenRouter AI tool schemas (`SOCIALSCRAPE_TOOLS`).            │
│  4. Executes plugin CLI via `subprocess.run(..., shell=False)`.        │
└────────────────────────────────────────────────────────────────────────┘
```

### Safety Rules:
1. **`shell=False` Strict Enforcement**: Subprocesses MUST always be executed using tokenized argument arrays (`[sys.executable, str(cli_path), ...]`) to prevent shell injection vulnerabilities.
2. **Crash Containment**: If a third-party platform API changes or throws an unhandled error, only the child process terminates; the master TUI REPL remains open, clean, and interactive.
3. **Collision-Free Direct Filesystem Discovery**: Do not rely on bare `import plugins` or `pkgutil.iter_modules()`, which fail in shared virtual environments (where multiple PPA tools define top-level `plugins`) and silently ignore hyphenated spoke folders (e.g. `gmaps-scraper`). The Hub's `PluginLoader` must scan `get_plugins_dir()` directly on the filesystem and dynamically register modules via `importlib.util.spec_from_file_location`, aliasing both hyphenated and underscore identifiers.
4. **Strict Single Canonical Slash Command & Kebab-Case Naming**:
   - Each spoke plugin MUST define exactly **ONE** canonical slash command in `plugin.json` under `"command"` (e.g. `"/youtube"`, `"/instagram"`).
   - Multi-word commands MUST use two words separated by a single hyphen (`/resume-batch`), NEVER underscores (`/resume_batch` ❌) and NEVER concatenated words (`/resumebatch` ❌).
   - Spoke plugins must NEVER register duplicate alias commands (e.g. `/facebook`, `/ads`, `/pin`, `/r`, `/th`, `/x`) into the central Hub autocomplete dropdown. The Hub registers strictly 1 canonical command per spoke in `_metadata`. Any aliases must be flagged as silent routing-only fallback (`is_alias=True`) to prevent dropdown pollution and duplicate command confusion.

---

## 10. Pattern 8: Defensive Sandboxing & LLM Prompt Injection Defense

1. **Path Traversal Jail-Checks**: Output file writers must verify that resolved file targets reside strictly within the permitted repository or user-designated output directories.
2. **XML Transcript Sandboxing**: Untrusted social media text (user captions, spoken transcripts, comments) must be wrapped in XML containment tags (`<transcript>...</transcript>` or `<post_content>...</post_content>`) before being returned to LLM reasoning loops to protect against prompt injection attacks.
3. **SSRF Filtering**: URL validators check input links against strict platform domain allowlists (`*.youtube.com`, `*.instagram.com`, `*.tiktok.com`, `*.linkedin.com`, `*.twitter.com`, `*.x.com`).

---

## 11. Pattern 9: Canonical Output Schema Whitelist & Field Locking Standard

Spoke plugins frequently enrich records that are fed into shared Hub outputs or master ledgers (`company.csv`, `people.csv`). To prevent tabular corruption and column spillover:

1. **Entity Dataclass Serialization (`to_dict()`)**:
   - Spoke dataclasses/models (`BusinessLead`, `CompanyFootprint`, `PersonLead`) MUST implement `to_dict()` methods that strictly filter their output against predefined canonical column schemas (e.g. `CANONICAL_COMPANY_FIELDS`, `CANONICAL_PEOPLE_FIELDS`).
   - Internal generator fields, split taxonomy components (`industry_division`, `industry_category`), and split geography components (`country`, `state_city`) must be merged into their canonical unified equivalents (`industry`, `location`) and omitted from the returned dictionary.
2. **The Unified Social Link Requirement**:
   - Spoke plugins must never inject disparate social media keys (`linkedin_url`, `twitter_url`, `instagram_url`, etc.) into shared company or people records.
   - All social URLs must be formatted into the unified `social_links` field via standard helpers (`format_unified_social_links()` or `parse_unified_social_links()`).
3. **Strict Whitelist Writing**:
   - Spoke storage helpers and exporters MUST enforce whitelist filtering before serialization (`{k: v for k, v in record.items() if k in target_fields}`). Unknown keys must never be appended dynamically to CSV headers.

---

## 12. Pattern 10: Hierarchical Multi-Tier Entity Decomposition & Multi-Mode Region Resolvers

In scrapers, crawlers, and geographic/taxonomic intelligence tools, execution targets often follow nested parent-child hierarchies (e.g. Country -> State/Region -> City/Business). To prevent cross-boundary corruption and give operators granular execution controls:

1. **Decoupled 3-Tier Resolution Workflow**:
   - **Tier 1 (Macro Entity)**: Operator selects high-level domains, countries, or divisions (e.g. US, Canada, UK).
   - **Tier 2 (Per-Parent Sub-Region Resolution)**: For *each* macro entity independently, prompt the operator with 3 distinct options:
     - `[1] All sub-regions [DEFAULT]`: Resolves 100% of child locations.
     - `[2] Multi-select checklist with entity counts & sample previews`: Renders an interactive checklist showing region name, total items, and sample elements (e.g. `North Carolina (48 cities: Charlotte, Raleigh, Greensboro...)`).
     - `[3] Uniform Random N sampling`: Randomly samples $N$ unique sub-regions without cross-border mixing.
   - **Tier 3 (Leaf Execution Targets)**: Compiles the final flat query list for execution.

2. **Polymorphic Backend Resolver Contract**:
   The engine resolver (e.g. `load_locations_from_countries()`) must accept multiple configuration types seamlessly:
   - `None`, `"all"`, or empty dict: defaults to 100% of entities.
   - `List[str]` or `Set[str]`: strictly filters to matching sub-regions case-insensitively.
   - `int`: randomly samples $N$ unique sub-regions and retrieves all child entities.
   - `Dict[str, Any]`: structured machine-actionable configuration (e.g. `{"mode": "manual", "states": [...]}` or `{"mode": "random", "count": N}`).

3. **Validation & Headless Interoperability**:
   Pydantic config models must type region parameters flexibly (e.g. `states_per_country: Dict[str, Any] = Field(default_factory=dict)`) so both interactive CLI wizards and automated AI agent workflows share the exact same underlying resolver.

---

## 13. Standardized `Outputs/` Subfolder Hierarchy

To keep workspace directories organized, all plugin output artifacts must be directed into one of the four standardized subdirectories:

| Subdirectory | Target Content | Supported File Types |
| :--- | :--- | :--- |
| `Outputs/videos/` | Downloaded full videos, reels, clips, and extracted audio tracks | `.mp4`, `.m4a`, `.mp3` |
| `Outputs/thumbnails/` | Video cover art, photo posts, carousels, and profile pictures | `.jpg`, `.webp`, `.png` |
| `Outputs/comments/` | Structured comment threads, replies, and sentiment objects | `.json` |
| `Outputs/text_posts/` | Transcripts, biographies, tweets, and Markdown profile summaries | `.txt`, `.json`, `.md` |

---

## 14. Checklist for Building New SocialScrape Plugins

When creating a new plugin (e.g. `plugins/reddit-scrape/` or `plugins/meta-ad-library/`), verify all 12 checklist items:

- [ ] **1. Scaffolding**: Create plugin folder with all 8 standard components (`core.py`, `cli.py`, `plugin.json`, `SKILL.md`, `README.md`, `requirements.txt`, `.gitignore`, `Outputs/`).
- [ ] **2. Pure SDK**: Implement `core.py` with pure functions (zero `print()`, zero `sys.exit()`, returns typed dicts).
- [ ] **3. Presenter CLI**: Implement `cli.py` with `argparse`, `-i` interactive wizard, and `--stdout` support.
- [ ] **4. Session Manager**: Integrate `RateLimiter` and `SessionManager` from `plugins/shared/session_manager.py` for authenticated endpoints.
- [ ] **5. Media Downloader**: Use `yt-dlp` format selectors for video and thumbnail downloads.
- [ ] **6. Standard Outputs**: Route all generated files into `Outputs/videos/`, `thumbnails/`, `comments/`, or `text_posts/`.
- [ ] **7. 7-Section README**: Write complete documentation adhering to the 7-section blueprint.
- [ ] **8. SKILL Protocol**: Document tool parameters and prompt sandboxing rules in `SKILL.md`.
- [ ] **9. Hub Integration**: Add slash command handler, feature matrix table, and OpenRouter tool schema to `socialscrape_cli.py`.
- [ ] **10. Root Linking**: Add plugin dependency reference (`-r plugins/<name>/requirements.txt`) to root `requirements.txt` and register skill in root `SKILL.md`.
- [ ] **11. Schema Whitelist**: Enforce canonical column whitelist in `to_dict()` and storage writers (zero unwhitelisted column spillover, unified `social_links`).
- [ ] **12. Non-Destructive Stepper**: Interactive wizards support backward step navigation via `'b'` or `'left'` with full input state retention.

---

## 15. Spoke Skill Specification & Root Hub Registration


Every spoke plugin MUST provide a valid `SKILL.md` adhering to the following structure:

1. **YAML Frontmatter**:
   - `name`: Exact plugin directory name (e.g. `instagram-scrape`).
   - `description`: Concise summary of extraction and downloading capabilities.
   - `triggers`: High-signal URL domains and keyword patterns that activate this skill.
2. **Execution Modes**:
   - In-Memory / Zero-Disk streaming (`python plugins/<name>/cli.py "<URL>" --stdout`).
   - Single and batch disk saves to `Outputs/`.
   - Interactive wizard invocation (`-i`).
3. **Hub Rollup & Dual-Mode Discovery**:
   - **Root Manifest**: The spoke is registered in the Hub's root `SKILL.md` Universal Command Matrix.
   - **CLI Stream**: The spoke's skill is streamed directly via `<tool> --skill <spoke-name>`.
   - **Global Materialization**: When installed globally, `bootstrap_global_storage()` mirrors `plugins/<name>/SKILL.md` to `~/.<tool>/skills/<name>/SKILL.md` so agents have offline filesystem access.
