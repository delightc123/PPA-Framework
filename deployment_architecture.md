# Deployment Architecture: Global Install & Dual-Mode Distribution 🚀📦

> **Document Type**: Deployment & Distribution Architectural Specification  
> **Version**: 1.0.0  
> **Location**: `.framework/PPA/deployment_architecture.md`

> **PPA Framework**: Cross-cutting — packaging, distribution & storage resolution for both Hub and Spokes.
> **Upstream (Root)**: [Plugin Project Architecture](plugin_project_architecture.md)
> **Related**: [CLI/TUI Hub Architecture](cli_architecture.md) — `pyproject.toml` packaging standard for the Hub this document distributes · [Plugin SDK Specification](plugin_sdk_specification.md) — how plugin layout & `Outputs/` resolve in global vs local mode · [Bulk Scraper Engine Specification](bulk_scraper_engine_specification.md) — `batches/` checkpoints in the storage layout · [agent.md](agent.md) — framework map

---

## 1. Executive Summary

SocialScrape supports **two execution modes** via a single codebase:

1. **Global Mode (End Users)**: Installed system-wide via `pip install`. The user types `socialscrape` from any terminal, anywhere on their machine. All data lives in `~/.socialscrape/`.
2. **Local Mode (Developers)**: Run directly from the cloned repo via `socialscrape-dev` or `python cli.py`. All data lives in the project directory (`CWD`).

The toggle between modes is a single environment variable: `SOCIALSCRAPE_LOCAL_MODE`.

---

## 2. Dual-Mode Architecture

```text
┌──────────────────────────────────────────────────────────────────────┐
│                     pyproject.toml Entry Points                      │
├──────────────────────────┬───────────────────────────────────────────┤
│  socialscrape            │  socialscrape-dev                         │
│  → main()                │  → main_dev()                             │
│  → Global Production     │  → Local Development                     │
│  → Reads ~/.socialscrape │  → Sets SOCIALSCRAPE_LOCAL_MODE=1         │
│                          │  → Reads project CWD                     │
└──────────────────────────┴───────────────────────────────────────────┘
```

### Entry Point Definitions (`pyproject.toml`)
```toml
[project.scripts]
socialscrape = "socialscrape_cli:main"
socialscrape-dev = "socialscrape_cli:main_dev"
```

### Mode Toggle (`socialscrape_cli.py`)
```python
def main():
    """Entry point for the global socialscrape command."""
    # Default: Global mode (reads from ~/.socialscrape/)
    run_tui_loop()

def main_dev():
    """Entry point for local development/testing mode."""
    os.environ["SOCIALSCRAPE_LOCAL_MODE"] = "1"
    main()
```

---

## 3. Storage Path Resolution (`storage.py`)

The `storage.py` module is the single source of truth for all path resolution across the entire project. Every plugin and the Hub CLI import paths from here.

```python
import os
from pathlib import Path

def get_root_dir() -> Path:
    """Returns the active root directory based on execution mode."""
    if os.environ.get("SOCIALSCRAPE_LOCAL_MODE") == "1":
        return Path.cwd()                      # Developer: project directory
    return Path.home() / ".socialscrape"        # End user: ~/.socialscrape/

def get_plugins_dir() -> Path:
    """Returns the plugins directory (bundled inside the package)."""
    return Path(__file__).parent.resolve() / "plugins"

def get_outputs_dir(platform: str) -> Path:
    """Returns the namespaced Outputs directory for a specific platform."""
    if os.environ.get("SOCIALSCRAPE_LOCAL_MODE") == "1":
        plugin_map = {
            "youtube": "youtube-scrape",
            "instagram": "instagram-scrape",
            "tiktok": "tiktok-scrape",
            "linkedin": "linkedin-scrape",
            "twitter": "twitter-scrape",
        }
        return Path(__file__).parent.resolve() / "plugins" / plugin_map[platform] / "Outputs"
    return get_root_dir() / "Outputs" / platform

def get_chats_dir() -> Path:
    """Returns the session history directory."""
    return get_root_dir() / "chats"

def get_myfiles_dir() -> Path:
    """Returns the user's personal workspace directory."""
    return get_root_dir() / "myfiles"

def get_env_file() -> Path:
    """Returns the .env configuration file path."""
    return get_root_dir() / ".env"

def get_config_path() -> Path:
    """Returns the path to the global config file."""
    return get_root_dir() / "config.json"
```

### Critical Design Decision: `get_plugins_dir()`
Plugins are **bundled inside the installed package** (they ship with the source code). The plugin directory is always resolved relative to the installed `storage.py` file location (`Path(__file__).parent / "plugins"`), NOT relative to `CWD`. This means:
- **Global mode**: Plugins run from `site-packages/socialscrape/plugins/`
- **Local mode**: Plugins run from the project's `plugins/` directory

This is the same physical directory in both modes (because `pip install -e .` symlinks the project).

---

## 4. Global Storage Layout (End User's Machine)

When an end user runs `socialscrape` for the first time, `bootstrap_storage()` automatically creates this structure in their home directory:

```text
~/.socialscrape/                        ← Auto-created on first launch
├── SKILL.md                            # Global Master Agent Skill & Universal Command Matrix
├── skills/                             # Materialized Spoke Skills for offline AI reading
│   ├── youtube-scrape/SKILL.md
│   ├── instagram-scrape/SKILL.md
│   ├── tiktok-scrape/SKILL.md
│   ├── linkedin-scrape/SKILL.md
│   ├── twitter-scrape/SKILL.md
│   ├── reddit-discussion-scraper/SKILL.md
│   ├── threads-scraper/SKILL.md
│   ├── pinterest-scraper/SKILL.md
│   └── meta-ad-library-scraper/SKILL.md
├── config.json                         # Global user preferences & settings
├── .env                                # User's API keys (OpenRouter, etc.)
│
├── chats/                              # AI agent session history logs
│   └── session_20260827_143022.json
│
├── myfiles/                            # User's personal workspace
│   └── (empty — user puts whatever they want here)
│
└── Outputs/                            # Downloaded media & extracted data
    ├── youtube/
    │   ├── videos/
    │   ├── thumbnails/
    │   ├── comments/
    │   └── text_posts/
    ├── instagram/
    │   ├── videos/
    │   ├── thumbnails/
    │   ├── comments/
    │   └── text_posts/
    ├── tiktok/
    │   ├── videos/
    │   ├── thumbnails/
    │   ├── comments/
    │   └── text_posts/
    ├── linkedin/
    │   └── text_posts/
    └── twitter/
        ├── videos/
        ├── thumbnails/
        ├── comments/
        └── text_posts/
```

### `config.json` Default Schema
```json
{
    "default_quality": "1080p",
    "default_output_format": "mp4",
    "openrouter_model": "cohere/north-mini-code:free",
    "agent_mode_default": true
}
```

---

## 5. Local Developer Layout (Unchanged)

When a developer clones the repo and runs `socialscrape-dev`, all paths resolve to the project's CWD. The developer's local structure stays exactly as-is:

```text
socialscrape/                           ← Project root = CWD
├── socialscrape_cli.py                 # Master TUI Engine
├── storage.py                          # Path resolver (NEW)
├── cli.py                              # Local CLI runner shortcut
├── pyproject.toml                      # Packaging config
├── requirements.txt                    # Root aggregated dependencies
├── .env                                # Developer API keys
├── .gitignore                          # Exclusion rules
├── README.md                           # Master documentation
├── SKILL.md                            # Master Hub Agent Skill Manifest (Pattern 4)
├── memory.md                           # Architectural decision records
│
├── myfiles/                            # Developer's architectural docs & tasks
│   ├── plugin_architecture/
│   │   ├── plugin_project_architecture.md
│   │   ├── plugin_sdk_specification.md
│   │   ├── deployment_architecture.md
│   │   ├── agentic_cli_architecture.md
│   │   └── cli_architecture.md
│   └── tasks/
│
├── chats/                              # AI session logs
│
├── Outputs/                            # Root-level fallback outputs
│
└── plugins/                            # THE SPOKES (unchanged)
    ├── shared/
    ├── youtube-scrape/
    ├── instagram-scrape/
    ├── tiktok-scrape/
    ├── linkedin-scrape/
    └── twitter-scrape/
```

---

## 6. One-Line Installation Commands & Deployment Standards

### 6.1 System Prerequisites & 1-Command Verification

Before installing any PPA project globally, the user's operating system must have two foundational runtime tools:
1. **Python 3.11+** (with "Add Python to PATH" enabled)
2. **Git** (required because pip invokes `git clone` under the hood to pull from GitHub)

#### Verification Commands:
Users or setup scripts can verify system readiness from any terminal:
```bash
python --version   # Must report Python 3.11 or higher
git --version      # Must report git version 2.x or higher
```

#### 1-Command Prerequisite Installation (If Missing):
- **Windows (PowerShell):**
  ```powershell
  winget install Python.Python.3.11 Git.Git
  ```
  *(Important: Restart terminal window after installation so Windows registers the updated PATH environment).*
- **macOS (Terminal):**
  ```bash
  brew install python git
  ```
- **Linux (Ubuntu / Debian):**
  ```bash
  sudo apt update && sudo apt install -y python3 python3-pip git
  ```

---

### 6.2 For End Users (Global 1-Line Install)

Users run a single command from their terminal without cloning the repository, installing the tool and all spoke dependencies globally into their environment:

```powershell
# Recommended: Bypasses Windows launcher stubs & PATH conflicts
python -m pip install git+https://github.com/<org>/<repo>.git

# Alternative (standard):
pip install git+https://github.com/<org>/<repo>.git
```

#### The `python -m pip` Anti-Launcher Crash Pattern (Architectural Rule):
> [!IMPORTANT]
> **Why PPA mandates `python -m pip` over bare `pip`:**  
> On Windows, standalone executable launchers in `Scripts/` (like `pip.exe`) contain a hardcoded shebang path pointing to the exact Python binary that created them. If an end user has upgraded, uninstalled, or moved Python versions, an orphaned `pip.exe` stub remaining on the system PATH will crash with:  
> `Fatal error in launcher: Unable to create process using '"C:\PythonXXX\python.exe" ...': The system cannot find the file specified.`  
> Invoking via `python -m pip` asks the currently active, healthy Python interpreter to execute its own bundled `pip` module directly, making installation 100% resilient across all operating systems and Python management tools (uv, py launcher, pyenv, standard python.org).

After installation, the user simply runs from ANY terminal:
```powershell
<tool-name>
```
*(If `<tool-name>` shortcut is not yet in the user's terminal PATH on Windows, they can also launch with `python -m <tool_name>_cli`)*.

On first launch, `bootstrap_global_storage()` automatically provisions the global user environment (`~/.<tool-name>/` or OS data directory) with `Outputs/` per spoke, `chats/`, `myfiles/`, default config, and AI model configurations.

---

### 6.3 Data Preservation & Non-Destructive Update Guarantee

A fundamental architectural requirement of the PPA Framework is **strict decoupling of executable code from user data**:

1. **Isolation of Code and State**:
   - `pip` installs executable Python packages strictly into Python's internal `site-packages/` directory.
   - All collected leads, scraped media, output tables, custom input CSVs, API keys (`.env`), and session logs are stored strictly in the user's home data directory: `~/.<tool-name>/` (e.g. `~/.gscrape/outputs/` or `~/.socialscrape/Outputs/`).
2. **Zero Data Loss During Updates**:
   - When a user updates the tool via:
     ```powershell
     python -m pip install --upgrade git+https://github.com/<org>/<repo>.git
     ```
     or runs the internal `/update` command, `pip` strictly upgrades the codebase in `site-packages/`. It **NEVER touches, overwrites, or deletes** existing output files in `~/.<tool-name>/outputs/`.
3. **Non-Destructive Bootstrapping**:
   - `bootstrap_global_storage()` uses `mkdir(parents=True, exist_ok=True)`. Existing user directories and CSV databases are preserved. The bootstrapper only synchronizes documentation manifests (`~/.<tool-name>/SKILL.md` and `skills/<spoke>/SKILL.md`), keeping agent tool schemas fresh while leaving user datasets intact.

---

### 6.4 For Developers (Local Install)

```powershell
git clone https://github.com/<org>/<repo>.git
cd <repo>
pip install -e .
<tool-name>-dev
```

The `-e` (editable) flag means code changes take effect immediately without reinstalling, and `<tool-name>-dev` isolates execution to the project's local workspace.

---

## 7. Self-Healing & Update Commands

The Hub CLI includes built-in maintenance and diagnostic commands:

| Command | Description |
| :--- | :--- |
| `/doctor` | Comprehensive 5-point environment, storage, browser, and AI diagnostic check |
| `/status` | Alias for `/doctor` |
| `/skill` | Render or inspect Master Hub SKILL.md or Spoke Plugin SKILL.md |
| `/skills` | List all active agent skills, trigger patterns, and command bindings |
| `/update` | Dual-mode updater: `git pull` (Local Mode) or `pip install --upgrade` (Global Mode) |
| `/fix` | Self-healing: Rebuilds storage folders, verifies browser binaries & repairs dependencies |
| `/uninstall` | Displays clean uninstall instructions |
| `/config` | Interactive configuration editor for `config.yaml` / `config.json` |

### Dual-Mode `/update` Implementation Pattern
The update command must dynamically detect whether it is executing inside a git clone or an installed global wheel:

```python
def handle_update(app=None, **kwargs) -> None:
    """Safely updates GSCRAPE preserving all user outputs and private chats."""
    is_git = (get_root_dir() / ".git").exists()
    
    if is_git:
        console.print("[cyan]Git repository detected. Pulling latest commits...[/cyan]")
        subprocess.check_call(["git", "pull", "origin", "main"])
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-e", "."])
    else:
        console.print("[cyan]Global installation detected. Upgrading via pip...[/cyan]")
        repo_url = "git+https://github.com/<org>/<tool-repo>.git"
        subprocess.check_call([sys.executable, "-m", "pip", "install", "--upgrade", repo_url])
    
    console.print("[bold green]✅ Update complete! Please restart the CLI.[/bold green]")
```

### System Diagnostics Protocol (`/doctor` & `/status`)
The `/doctor` command performs non-destructive, sub-second inspection across 5 vital subsystems:
1. **Execution Mode & Runtime**: Reports `GLOBAL` vs `LOCAL`, active Python version, executable binary path, and package version.
2. **Storage & Inputs**: Verifies write permissions to `~/.<app>/`, checks presence of `config.yaml`, `.env`, and counts pre-seeded location/industry CSV registries.
3. **Master Lead Databases**: Inspects record counts in relational spreadsheets (`company.csv`, `people.csv`).
4. **Stealth Browser Engine**: Queries `camoufox.pkgman` to confirm if stealth browser binaries exist on disk.
5. **AI Intelligence Core**: Verifies OpenRouter API key configuration (safely masked: `sk-or-v1-****...a1b2`) and active model name.

---

## 8. Source Code Visibility & Security Model

### What End Users Receive
When installed via `pip install`, the user receives the runnable Python source files in their `site-packages/` directory. **Python does not compile to binary** — the `.py` files are readable.

### What End Users Do NOT Receive
- `.git/` directory (no commit history or branch access)
- `myfiles/tasks/` (developer task logs and walkthroughs)
- Developer-specific architectural docs (these are git-ignored via the global install)
- Direct repository access (if the repo is private)

### Protection Layers
1. **Private GitHub Repository**: Only explicitly invited collaborators or PAT holders can install.
2. **No Development Artifacts**: Build manifests exclude task logs, audit trails, and internal docs.
3. **Operational Abstraction**: Users interact via the TUI — they never need to open a code editor.

> **Important**: If full source code concealment is required in the future, the project would need to compile to standalone binaries using PyInstaller or distribute as a SaaS API. The current model (pip package distribution) does NOT hide source code.

---

## 9. Package Configuration (`pyproject.toml`)

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "socialscrape"
version = "2.0.1"
description = "Universal Social Media Data Extraction & Downloader TUI-CLI Hub"
readme = "README.md"
requires-python = ">=3.9"
license = { text = "MIT" }
dependencies = [
    "rich>=13.0.0",
    "prompt-toolkit>=3.0.0",
    "art>=6.0",
    "requests>=2.28.0",
    "yt-dlp>=2024.0.0",
    "youtube-transcript-api>=0.6.0",
    "python-dotenv>=1.0.0",
    "platformdirs>=3.0.0"
]

[project.scripts]
socialscrape = "socialscrape_cli:main"
socialscrape-dev = "socialscrape_cli:main_dev"

[tool.setuptools]
py-modules = ["socialscrape_cli", "cli", "storage", "ui", "agent", "batch_manager", "handlers"]

[tool.setuptools.package-data]
"*" = ["*.json", "*.md", "*.txt"]

[tool.setuptools.packages.find]
include = ["plugins*"]
```

> **Packaging Mandate**: In flat package layouts, `py-modules` must explicitly enumerate **all** root modules. In addition, `[tool.setuptools.package-data]` must be defined with `"*" = ["*.json", "*.md", "*.txt"]` so that plugin manifests (`plugin.json`) and agent protocols (`SKILL.md`) are bundled inside the global distribution wheel.

---

## 10. Bootstrap Sequence (First Launch)

```text
User types `socialscrape` →
    main() →
        storage.bootstrap_global_storage() →
            1. Resolve OS root dir (platformdirs / ~/.socialscrape/)
            2. Materialize SKILL.md and skills/<spoke>/SKILL.md in user data dir
            3. Create config.json with defaults (if missing)
            4. Create .env template (if missing)
            5. Create chats/ directory (if missing)
            6. Create myfiles/ directory (if missing)
            7. Create Outputs/<platform>/<subfolder> for all 9 platforms (if missing)
        →
        run_tui_loop() →
            TUI launches with full functionality
```

---

## 11. Zero-Prerequisite Stealth Browser Binary Self-Healing (`storage.py`)

For tools that depend on stealth anti-detect browsers (e.g., `camoufox`, Playwright), asking non-technical users to manually execute secondary terminal commands (like `camoufox fetch` or `playwright install`) introduces significant onboarding friction and leads to runtime crashes if skipped.

### The Self-Healing Lifecycle
The PPA standard mandates that the browser launcher must proactively verify binary presence and auto-fetch missing binaries directly inside Python before attempting initialization:

```python
def is_browser_installed() -> Tuple[bool, Optional[str], Optional[str]]:
    """Checks if stealth browser binaries exist on local disk without side effects."""
    try:
        from camoufox.pkgman import launch_path, installed_verstr
        p = launch_path()
        v = installed_verstr()
        if p and Path(p).exists():
            return True, str(v) if v else None, str(p)
        return False, None, None
    except Exception:
        return False, None, None

def ensure_browser_installed() -> bool:
    """Proactively downloads missing browser binaries with rich terminal progress."""
    installed, _, _ = is_browser_installed()
    if installed:
        return True

    from rich.console import Console
    Console().print("[bold cyan]🦊 Stealth browser binaries not detected. Auto-fetching now...[/bold cyan]")
    
    try:
        res = subprocess.run([sys.executable, "-m", "camoufox", "fetch"], capture_output=True, text=True)
        installed, _, _ = is_browser_installed()
        return installed
    except Exception as e:
        logger.error(f"Auto-fetch failed: {e}")
        return False
```

### Integration into `BrowserManager.launch()`
```python
@contextmanager
def launch(self, headless: Optional[bool] = None) -> Generator[Any, None, None]:
    """Guarantees browser binary readiness before instantiating browser context."""
    ensure_browser_installed()
    is_headless = self.headless if headless is None else headless
    with Camoufox(headless=is_headless) as browser:
        yield browser
```
This guarantees that `pip install git+https://...` is truly a **single-command global installation** with zero manual prerequisites.

---

## 12. Agent-Native Global Distribution & Headless Introspection (`--skill`)

### The Packaging Boundary Problem
When a Python package is installed globally (`pip install`), code and package data (`[tool.setuptools.package-data] "*" = ["*.json", "*.md", "*.txt"]`) are extracted to Python's internal `site-packages/` directory. For external AI coding agents (such as Codex, Claude Code, Antigravity) running inside arbitrary user projects, `site-packages` is completely hidden and inaccessible.

### The PPA Solution: Dual-Tier Materialization & Zero-Disk Streaming
To guarantee that tools remain 100% agent-native in global environments:
1. **Materialization into User Storage (`~/.<tool>/SKILL.md` & `skills/`)**: On launch or upgrade, `bootstrap_global_storage()` copies the Root `SKILL.md` to `~/.<tool>/SKILL.md` and copies each spoke plugin's `SKILL.md` into `~/.<tool>/skills/<spoke>/SKILL.md`. Agents inspecting the user directory can read the complete skill hierarchy offline.
2. **Headless CLI Introspection (`<tool> --skill [spoke]`)**: Any agent with terminal access can run:
   - `<tool> --skill`: Dumps the Master Hub `SKILL.md` directly to stdout for immediate LLM parsing.
   - `<tool> --skill <spoke>`: Dumps a specific spoke plugin's `SKILL.md` directly to stdout.
   - `<tool> --skills`: Dumps a summary table of all active plugins, commands, and trigger keywords.
This eliminates all onboarding friction, making globally installed PPA tools instantly usable by any AI agent.

