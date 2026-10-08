# Universal CLI & TUI Hub Architecture Specification 🌐

> **Pattern Purpose**: A standardized architectural blueprint for building modern, high-fidelity terminal user interfaces (TUI) and global CLI hubs for modular plugin projects (inspired by OpenCode).
> **Agentic Specification**: For the complete guide on AI agent integration, multi-turn session lifecycle (`/resume`, `/clear`, `/new`, `/delete`), OpenRouter tool calling, and conversational UI rendering, see [agentic_cli_architecture.md](agentic_cli_architecture.md).

> **PPA Framework**: Hub layer — the Universal CLI/TUI standard for the Root Hub.
> **Upstream (Root)**: [Plugin Project Architecture](plugin_project_architecture.md)
> **Extends into**: [Agentic CLI Architecture](agentic_cli_architecture.md) — AI agent mode, session lifecycle & tool calling on top of this TUI standard
> **Related**: [Deployment Architecture](deployment_architecture.md) — packaging & dual-mode entry points for this Hub · [Bulk Scraper Engine Specification](bulk_scraper_engine_specification.md) — the batch/resume engine behind `/batch` & `/resume-batch` · [agent.md](agent.md) — framework map

---

## 1. Overview & Architectural Role

In the Hub-and-Spoke model, the **Root CLI/TUI Hub** serves as the unified presentation, discovery, session memory, and execution layer for all modular plugins in `plugins/`.

```text
                                  ┌───────────────────────────────┐
                                  │   Global Terminal Command     │
                                  │       (`socialscrape`)        │
                                  └───────────────┬───────────────┘
                                                  │
                                                  ▼
                                  ┌───────────────────────────────┐
                                  │       Root TUI Engine         │
                                  │    (`socialscrape_cli.py`)    │
                                  │ ───────────────────────────── │
                                  │  • OpenCode 3-Row Block Logo  │
                                  │  • Persistent REPL Autocomplete│
                                  │  • Session Memory & Logs      │
                                  │  • Command Palette Modal      │
                                  │  • Docked Footer Status Bar   │
                                  └───────────────┬───────────────┘
                                                  │
                      ┌───────────────────────────┴───────────────────────────┐
                      ▼                                                       ▼
       ┌──────────────────────────────┐                       ┌──────────────────────────────┐
       │     Internal Hub Actions     │                       │  Subprocess Plugin Runners   │
       │ ──────────────────────────── │                       │ ──────────────────────────── │
       │  • `/new`    (Fresh Session) │                       │  • `/youtube`                │
       │  • `/resume` (Replay Modal)  │                       │  • `/tiktok`                 │
       │  • `/clear`  (Clean History) │                       │  • `/meta`                   │
       │  • `/delete` (Purge Logs)    │                       │  (Runs in isolated process)  │
       │  • `/skill`  (Agent Manifest)│                       └──────────────────────────────┘
       │  • `/plugins` (Feature Drill)│
       │  • `/outputs` (Tree View)    │
       │  • `/update`  (Safe Git Pull)│
       │  • `/fix`     (Pip Repair)   │
       └──────────────────────────────┘
```

---

## 2. Packaging & Root Namespace Isolation

### A. Flat Root Packaging (`pyproject.toml`)
To allow users to install and run the tool globally (`pip install -e .`) without requiring nested redundant subfolders:

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "socialscrape"
version = "1.0.0"
description = "Universal Social Media Data Extraction & Downloader TUI-CLI Hub"
readme = "README.md"
requires-python = ">=3.9"
dependencies = [
    "rich>=13.0.0",
    "prompt-toolkit>=3.0.0",
    "art>=6.0",
    "requests>=2.28.0",
    "yt-dlp>=2024.0.0",
    "mcp>=1.0.0"
]

[project.scripts]
socialscrape = "socialscrape_cli:main"

[tool.setuptools]
py-modules = ["socialscrape_cli", "cli", "storage", "ui", "agent", "batch_manager", "handlers"]

[tool.setuptools.package-data]
"*" = ["*.json", "*.md", "*.txt"]
```

> **Packaging Mandate**: In flat root package structures, **every** root `.py` module utilized by the CLI must be explicitly declared in `py-modules`. Omitting modules causes Setuptools to exclude them from distribution wheels and non-editable installs, resulting in runtime `ModuleNotFoundError`. In addition, `[tool.setuptools.package-data]` must be defined with `"*" = ["*.json", "*.md", "*.txt"]` to guarantee that both the Root `SKILL.md` and Spoke `plugins/*/SKILL.md` are bundled inside the global distribution wheel.

### B. Avoiding VirtualEnv Namespace Collisions
- **Rule**: Global console scripts in Python must NEVER point to a generic `cli:main` if installed into a shared virtual environment (like Hermes, uv, or system pip), as another package's `cli.py` in `site-packages` will collide.
- **Best Practice**:
  1. Main TUI engine lives in `socialscrape_cli.py` (unique prefix).
  2. Root `cli.py` is a simple 4-line runner for `python cli.py`.
  3. `pyproject.toml` maps the global binary to `socialscrape_cli:main`.

---

## 3. OpenCode Visual Design Standard & Dynamic Theme Injection

The PPA Framework mandates **1:1 structural UX parity with OpenCode** while supporting **Dynamic Theme & Branding Injection** across different plugin ecosystems.

### A. Layout Structure (1:1 OpenCode Parity)
- **Centralized Container**: Bounded layout centered in the terminal.
- **Boxed Input Area**: Styled bottom border box for prompt input.
- **Drop-Down Autocomplete**: Autocomplete suggestions drop *downwards* directly below the active command line.
- **Floating Modals**: `/plugins`, `/models`, and `/resume` open rich, interactive floating overlays.

### B. Dynamic Theme & Logo Injection Pattern
While structural UI elements remain constant across all PPA tools, branding tokens are injected per-project:

| Token / Asset | **SocialScrape (Default)** | **GSCRAPE (Lead Gen)** | **Custom Tool Template** |
| :--- | :--- | :--- | :--- |
| **Primary Accent** | `#5c9cf5` (Agent Blue) | `#00ffff` (Cyan) | Project Brand Color |
| **Secondary Accent** | `#fab283` (Peach) | `#00aaaa` (Dark Cyan) | Secondary Brand Color |
| **Border Neutral** | `#3c3c3c` (Slate) | `cyan` / `dim cyan` | Subtle Neutral Tone |
| **Header Banner** | 3-Row Unicode Block | Compact Slant ASCII (<50 chars) | 3-Row Block or Compact Text |
| **Active Mode Glyph** | `  ┃ 🤖 > ` (`#5c9cf5`) | `  ┃ 🤖 > ` (`#00ffff`) | `  ┃ 🤖 > ` (Primary Brand) |

### C. Terminal Screen Buffer, Viewport Reset & Mouse Scroll Protocol
The PPA standard balances **1:1 visual OpenCode aesthetics** (clean, centered banner, shortcut table, docked toolbar) with **100% native terminal scrollability**.

#### 1. The Core Architectural Conflict
In standard terminal emulators (Windows Terminal, VS Code Terminal, xterm):
* **Alternate Screen Buffer (`\033[?1049h`)**: Provides a full-screen canvas, but **physically disables the terminal scrollbar and has ZERO lines of scrollback memory**. Any long outputs (e.g. `/doctor`, `/readme`, bulk scrape streams) pushing past the screen height are discarded and lost.
* **Alternate Scroll Mode (`\033[?1007h`) & Mouse Tracking (`\033[?1000h` / `\033[?1006h`)**: Intercepts mouse wheel events and converts them into **Cursor Up/Down keys**. In `prompt_toolkit`, this causes mouse scrolling to cycle through command history and type `/command` into the prompt input box instead of scrolling output! Furthermore, terminal mode escape sequences are **sticky per terminal tab** and will persist across process restarts unless explicitly cleared.
* **How OpenCode Solves This (OpenCode Architecture Reference)**: OpenCode is an OpenTUI application (`@opentui/core`) that renders into an internal `<ScrollBox>` component (`scroll: ScrollBoxRenderable`) in memory at 60 FPS. The terminal window itself never scrolls; OpenCode scrolls its internal component.

#### 2. The Proven PPA Standard (Primary Buffer + Viewport Reset by Default)
To achieve both clean visual presentation and 100% native scrollability without locking the terminal:

1. **Default: Primary Buffer with Viewport Reset (`\033[2J\033[H`)**:
   Instead of `\033[?1049h`, clear the visible viewport and home the cursor on launch. This clears the shell prompt (`PS C:\...>`) while keeping the terminal emulator's native scrollback memory 100% active.
2. **Mandatory Hardware Reset Hook (`reset_terminal_mouse_mode`)**:
   Always execute an explicit escape sequence at module load, startup, screen clears (`/clear`), and exit to purge any sticky mouse interception modes:
   ```python
   TERMINAL_MOUSE_RESET = "\033[?1049l\033[?1007l\033[?1000l\033[?1002l\033[?1003l\033[?1006l\033[?1015l\033[?25h"

   def reset_terminal_mouse_mode() -> None:
       """Forces the terminal emulator to unhook mouse interception and restore 100% native scrollback."""
       try:
           sys.stdout.write(TERMINAL_MOUSE_RESET)
           sys.stdout.flush()
       except Exception:
           pass
   ```
3. **PromptSession Configuration (`mouse_support=False`)**:
   The main `PromptSession` must be initialized with `mouse_support=False` so the terminal emulator handles mouse wheel scrolling natively.
4. **Persistent Command History (`FileHistory`)**:
   Always use `FileHistory(get_history_file())` so Up/Down arrows navigate previous command history across runs.
5. **In-App Scrollbox Viewer (`/history` / `PageUp` / `Ctrl+U`)**:
   When users want an immersive, focused full-screen scrollbox (matching OpenCode's internal scrollbox), launch a dedicated `prompt_toolkit.Application` with `full_screen=True` and `mouse_support=True`. Always execute `reset_terminal_mouse_mode()` in its `finally:` block.
6. **Opt-in Alternate Screen (`--alt-screen`)**:
   Provide `--alt-screen` as an opt-in CLI flag for environments where strict screen isolation without scrollback is explicitly desired.

7. **Exit Teardown & Clean Terminal Protocol (`cleanup_terminal_on_exit`)**:
   When the user exits the interactive TUI (via `Esc` on an empty buffer, double `Ctrl+C`, `/exit`, or `EOF`), the terminal must never leave trailing session clutter requiring the user to run `clear` manually. The `finally:` block must cleanly wipe the viewport and restore normal cursor/mouse states:
   ```python
   def cleanup_terminal_on_exit() -> None:
       """Neatly clears the terminal screen and resets mouse/cursor state on exit."""
       try:
           reset_terminal_mouse_mode()
           sys.stdout.write("\033[2J\033[H")
           sys.stdout.flush()
           if os.name == "nt":
               os.system("cls")
       except Exception:
           pass
   ```
8. **Clean Exit Keybindings (`escape` & `Ctrl+C`)**:
   - `escape`: If the input buffer contains text, clear it. If the buffer is already empty, cleanly exit the application (`event.app.exit(result=None)`).
   - `Ctrl+C`: Gracefully prompt on the first press and exit on the second, passing control directly to `cleanup_terminal_on_exit()`.

```python
def run(self) -> None:
    reset_terminal_mouse_mode()
    self.setup()

    alt_screen = "--alt-screen" in sys.argv
    if alt_screen:
        # Opt-in Alternate Screen Buffer
        sys.stdout.write("\033[?1049h\033[?1007h\033[2J\033[H")
    else:
        # Default: Clean Viewport Reset (100% Native Terminal Scrollback Preserved)
        reset_terminal_mouse_mode()
        sys.stdout.write("\033[2J\033[H")
    sys.stdout.flush()

    try:
        display_home_screen()
        # Interactive REPL Loop with FileHistory
        while self.running:
            prompt_val = self.session.prompt(...)
            if prompt_val is None:  # Triggered by Esc on empty buffer
                break
            ...
    finally:
        cleanup_terminal_on_exit()
```

### D. Precise Banner Centering & Version Alignment Formula
When centering a multi-line banner via `Align.center(banner)`, leading spaces before the version number must equal `logo_width - len(version_string)`. Any excess whitespace expands the layout bounding box, pulling the centered logo off-balance to the left:
```python
total_width = len(LOGO_LEFT[0]) + 1 + len(LOGO_RIGHT[0])
ver_str = f"v{__version__}"
banner.append(" " * (total_width - len(ver_str)) + ver_str + "\n", style=f"{COLOR_MUTED}")
```

### E. Uniform Intra-Word Character Spacing
In 2-tone block logos (e.g., `G` gray + `SCRAPE` cyan or `OPEN` gray + `CODE` white), maintain uniform 1-character spacing across all letters. Do not introduce extra gap spaces at the color junction.

---

## 4. TUI Autocomplete Engine & Feature Discovery

### A. Strict Slash Autocomplete (Drop-Down Style)
Autocompletions trigger strictly upon typing `/` and display downwards:
```python
class PersistentSlashCompleter(Completer):
    COMMANDS = [
        ("/agent",    "enable AI agent mode for natural language scraping"),
        ("/manual",   "switch to manual slash-command mode"),
        ("/help",     "show full command matrix & shortcuts"),
        ("/plugins",  "list active plugins & interactive feature details"),
        ("/models",   "browse & select from curated free OpenRouter AI models"),
        ("/youtube",  "transcripts, metadata, comments & 4K video"),
        ("/outputs",  "view scraped videos & media folders"),
        ("/new",      "start a fresh new chat session"),
        ("/clear",    "clear message history of current session"),
        ("/resume",   "interactive modal to resume saved session"),
        ("/delete",   "delete saved chat sessions from disk"),
        ("/readme",   "read project documentation in terminal"),
        ("/skill",    "view agent skill manifest & tool schemas"),
        ("/skills",   "list active plugin skills & trigger patterns"),
        ("/update",   "pull latest updates from GitHub safely"),
        ("/fix",      "repair & reinstall Python dependencies"),
        ("/exit",     "terminate SocialScrape session"),
    ]

    def get_completions(self, document: Document, complete_event: CompleteEvent):
        text = document.text_before_cursor
        if not text.startswith("/"):
            return
        if text == "/":
            for cmd, desc in self.COMMANDS:
                yield Completion(cmd, start_position=-len(text), display=cmd, display_meta=desc)
            return

        query = text.lower()
        for cmd, desc in self.COMMANDS:
            if cmd.lower().startswith(query) or (len(query) > 1 and query[1:] in cmd.lower()):
                yield Completion(cmd, start_position=-len(text), display=cmd, display_meta=desc)
```

### B. Interactive Feature Drilldown Pattern (`/plugins`)
To eliminate command sprawl and prevent feature bloat, plugin capabilities are inspected directly through an interactive selector within the `/plugins` command, rendering a full feature and subcommand matrix.

### C. Strict Slash Command Naming Convention & Zero-Duplicate Rule
To ensure a clean, professional, and collision-free CLI user experience, the PPA framework strictly enforces the following command design rules across all Hubs and Spokes:

1. **Strict Kebab-Case Naming (Two Words with a Hyphen)**:
   - All multi-word slash commands must use lowercase letters separated strictly by a hyphen (`-`).
   - **Correct**: `/resume-batch`, `/chat-history`, `/export-csv`.
   - **Incorrect**: `/resume_batch` (underscores strictly forbidden ❌), `/resumebatch` (concatenated words forbidden ❌).

2. **Single Canonical Command per Feature / Spoke (Zero Duplicates)**:
   - Each distinct capability or spoke plugin must expose **exactly ONE** canonical slash command in the Hub.
   - There must never be multiple visible commands performing the same action or aliases sharing the autocomplete dropdown (e.g., do NOT register both `/resume-batch` and `/resumebatch`, or `/models` and `/model`).
   - Each spoke plugin has exactly one canonical command matching its primary capability (e.g., `/youtube`, `/instagram`, `/meta`).

3. **Zero Alias Pollution in Autocomplete Dropdowns**:
   - Aliases (e.g. shorthand letters `/r`, `/th`, `/x`, or synonym keywords `/quit`, `/clean`, `/status`) must **NEVER** be registered in the autocomplete metadata or yielded by `PersistentSlashCompleter`.
   - If backward-compatible aliases are supported for user typing convenience, the Command Router must mark them as silent fallback aliases (`is_alias=True`). Silent aliases execute when explicitly typed into the prompt, but are strictly excluded from:
     - `PersistentSlashCompleter` autocomplete dropdowns.
     - Universal Help command tables (`/help`).
     - Command discovery metadata (`router.get_command_metadata()`).

---

## 5. Session Management Architecture (`/resume`, `/clear`, `/new`, `/delete`)

All conversational history and tool executions are persisted in `chats/<session_id>.json`:
- `/new`: Instantiates a fresh `session_<timestamp>.json` and resets in-memory history.
- `/clear`: Wipes active session memory and redraws a clean terminal.
- `/resume`: Displays an interactive modal table of previous sessions and visually replays the cards into the terminal.
- `/delete`: Interactively deletes individual session logs or purges all (`/delete all`).

---

## 6. Subprocess Fault-Isolation Pattern

### Why Subprocesses Instead of Dynamic Imports?
When building an interactive hub that executes sub-tools:
1. **Fault Isolation**: If a plugin crashes, throws an unhandled exception, or calls `sys.exit()`, only the child process dies. The main TUI catches the exit code and cleanly redraws the prompt.
2. **Buffer Safety**: Chaining multiple interactive `input()` or `prompt_toolkit` sessions in the same Python process corrupts the terminal buffer. Spawning `subprocess.run([sys.executable, plugin_cli, "-i"])` temporarily yields full console control to the plugin, and regains clean control when finished.

---

## 7. User Data Protection & Git Isolation

To ensure that running `/update` (`git pull`) never wipes or conflicts with local user outputs:

### `.gitignore` Architecture
```gitignore
# User Personal Workspace & Custom Task Files
myfiles/*
!myfiles/.gitkeep

# Plugin Scraped Media & Outputs (Isolated per machine)
plugins/*/Outputs/*
!plugins/*/Outputs/.gitkeep
Outputs/*
!Outputs/.gitkeep

# AI Chat Session Logs (Isolated per machine)
chats/*
!chats/.gitkeep

# Python Build Cache
*.egg-info/
dist/
build/
```

### `.gitkeep` Directory Retention
- Git cannot track empty folders.
- Placing a `.gitkeep` in `myfiles/`, `chats/`, and `plugins/*/Outputs/` ensures that a fresh `git clone` automatically initializes the directory tree for new users, while `.gitignore` ensures their downloaded files and private chat histories remain private and untracked.

---

## 8. Standardized Root Hub README Blueprint (8-Section Architecture)

While individual plugins follow the **7-Section Plugin README Blueprint** (defined in `plugin_sdk_specification.md`), the project's root `README.md` serves as the public entry point and global hub directory. Every PPA-compliant Root `README.md` must adhere to this **8-Section Standard**:

```markdown
# Section 1: ASCII Art Brand Banner & Badges
- Centered unicode block logo (<50 columns wide to prevent scrollback fracture)
- One-line italicized value proposition
- Shields.io badges: Version, Python version (>=3.9), Architecture (Hub-and-Spoke), License (MIT)

# Section 2: 📋 Prerequisites & ⚡ 1-Line Global Installation
- Prerequisites verification: `python --version` (>=3.11) and `git --version` (>=2.x)
- 1-Command installer for missing tools (`winget install Python.Python.3.11 Git.Git`, `brew install python git`, `apt install python3 python3-pip git`)
- Canonical global install command: `python -m pip install git+https://github.com/<org>/<repo>.git` (mandated over bare `pip` to avoid Windows launcher PATH conflicts)
- Universal launch command: `<tool-name>` (with fallback `python -m <tool_name>_cli`)
- Automated browser binary setup notice (e.g. Camoufox auto-fetch)
- Data preservation guarantee: updating via `python -m pip install --upgrade` or `/update` strictly updates `site-packages/` and never overwrites user output data in `~/.<tool-name>/Outputs/`
- Clear explanation of automatic first-launch storage provisioning (~/.<tool-name>/Outputs/, chats/, config.yaml/config.json, .env)

# Section 3: 🛠️ Developer Setup (Local Mode)
- Local clone instructions: `git clone`, `cd`, `pip install -e .`, `<tool-name>-dev`
- Explicit contrast: `<tool-name>` (Global Mode) vs `<tool-name>-dev` (Local Mode via `LOCAL_MODE=1`)

# Section 4: 🔄 Self-Updating Command
- In-TUI command: `/update`
- Terminal upgrade command: `pip install --upgrade git+https://github.com/<org>/<repo>.git`

# Section 5: 🖥️ The Interactive OpenCode-Style TUI
- Visual ASCII preview of the active TUI canvas
- Command Reference Table (Command, Description, Shortcut)

# Section 6: 📦 Active Spokes / Plugins Matrix
- Table listing all discovered plugins with Platform, Status, Primary Capabilities, and links to `plugins/<spoke>/README.md`

# Section 7: 🏗️ Architecture: Hub-and-Spoke Model
- ASCII architecture diagram demarcating Hub presentation layer from isolated Spokes in `plugins/`
- Four core architectural tenets: SDK Isolation, Dynamic Discovery, Dual-Mode Storage, and Alternate Screen Buffer with Scrollback

# Section 8: 🔒 Security & Data Rails
- Anti-bot burner management & rate limit safety
- Isolated local storage (.gitignored user files)
- License declaration
```

---

## 9. The Unified Command Router & Dispatcher Pattern (`handlers.py`)

In the PPA architecture, the CLI entry point (`gscrape_cli.py`) should NEVER contain giant, monolithic `if/elif/else` blocks dispatching user inputs. Instead, execution routing is cleanly abstracted into an extensible `CommandRouter`:

```python
class CommandRouter:
    """Routes CLI inputs and slash commands to appropriate handlers."""

    def __init__(self):
        self._handlers: Dict[str, Callable[..., Any]] = {}
        self._metadata: Dict[str, str] = {}
        self._register_default_commands()

    def register(self, command: str, handler: Callable[..., Any], description: str) -> None:
        cmd = command.lower().strip()
        if not cmd.startswith("/"):
            cmd = f"/{cmd}"
        self._handlers[cmd] = handler
        self._metadata[cmd] = description

    def dispatch(self, command_line: str, **kwargs) -> Any:
        parts = command_line.strip().split()
        if not parts:
            return None
        cmd = parts[0].lower()
        args = parts[1:]
        handler = self._handlers.get(cmd)
        if handler:
            return handler(args=args, **kwargs)
        return False
```

### Key Architectural Tenets of `CommandRouter`
1. **Case-Insensitive Slash Normalization**: Automatically prepends `/` and converts to lowercase.
2. **First-Class Aliases**: Canonical commands and convenience aliases point to identical handler callbacks (e.g. `/doctor` and `/status`, `/clean` and `/clear`).
3. **Persistent Autocomplete Integration**: The command router's `_metadata` dictionary directly feeds `prompt_toolkit`'s `PersistentSlashCompleter`. Unwanted aliases are cleanly excluded from dropdown popups to keep completions minimal:
```python
meta_dict = handlers.router.get_command_metadata()
filtered_meta = {k: v for k, v in meta_dict.items() if k not in EXCLUDED_ALIASES}
```

---

## 10. System Diagnostic Dashboard Presentation (`render_doctor_report`)

To provide immediate visibility into system health, environment state, and storage permissions, PPA Hubs provide an OpenCode-style diagnostic card via `/doctor`:

```python
def render_doctor_report(diagnostics: Dict[str, Any]) -> None:
    """Renders a standardized OpenCode 2-column system diagnostic inspection table."""
    table = Table(box=box.ROUNDED, border_style="#3c3c3c", show_header=True, padding=(0, 2))
    table.add_column("Component / Diagnostic", style="bold #eeeeee", width=30)
    table.add_column("Status & Details", style="#808080", width=50)

    # 1. Runtime & Execution Mode
    mode_text = "[bold green]GLOBAL MODE (~/.<app>/)[/bold green]" if diagnostics["is_global"] else "[bold yellow]LOCAL MODE (CWD)[/bold yellow]"
    table.add_row("Execution Mode", mode_text)
    table.add_row("Python Runtime", f"{diagnostics['python_version']} ({diagnostics['python_path']})")
    table.add_row("Package Version", f"v{diagnostics['package_version']}")
    table.add_row("", "")

    # 2. Storage & Input Registries
    table.add_row("[bold cyan]── Storage & Filesystem ──[/bold cyan]", "")
    table.add_row("Root Directory", str(diagnostics['root_dir']))
    table.add_row("Inputs Directory", f"{diagnostics['inputs_dir']} ({diagnostics['locations_count']} locations, {diagnostics['industries_count']} industries)")
    table.add_row("Outputs Directory", str(diagnostics['outputs_dir']))
    table.add_row("Chats Directory", f"{diagnostics['chats_dir']} ({diagnostics['chats_count']} saved sessions)")
    table.add_row("", "")

    # 3. Database Volume
    table.add_row("[bold cyan]── Lead Database Health ──[/bold cyan]", "")
    table.add_row("company.csv (24 cols)", f"[bold green]{diagnostics['company_records']:,}[/bold green] verified company records")
    table.add_row("people.csv (12 cols)", f"[bold green]{diagnostics['people_records']:,}[/bold green] decision-maker profiles")
    table.add_row("", "")

    # 4. Stealth Browser Engine
    b_status = f"[bold green]● INSTALLED[/bold green] (v{diagnostics['browser_version']})" if diagnostics['browser_installed'] else "[bold red]✖ MISSING (Auto-fetch enabled)[/bold red]"
    table.add_row("[bold cyan]── Stealth Browser Engine ──[/bold cyan]", "")
    table.add_row("Camoufox Anti-Detect", b_status)
    if diagnostics.get("browser_path"):
        table.add_row("Executable Path", str(diagnostics['browser_path']))
    table.add_row("", "")

    # 5. AI Intelligence Core
    k_status = f"[bold green]● Configured[/bold green] ({diagnostics['api_key_masked']})" if diagnostics['api_key_configured'] else "[bold yellow]○ Missing (Manual Mode only)[/bold yellow]"
    table.add_row("[bold cyan]── AI Intelligence Core ──[/bold cyan]", "")
    table.add_row("OpenRouter API Key", k_status)
    table.add_row("Active AI Model", f"[cyan]{diagnostics['active_model']}[/cyan]")

    panel = Panel(Align.center(table), title="[bold white]🩺 System Diagnostics (/doctor)[/bold white]", border_style="cyan", box=box.ROUNDED)
    console.print(Align.center(panel))
```

---

## 11. Non-Destructive Interactive Wizard Stepper & Bidirectional Navigation Pattern

### A. The Core UX Problem
Standard CLI wizards are strictly forward-only. If an operator reaches Step 4 or Step 5 (e.g. execution settings, filters, or batch parameters) and realizes they made an error on Step 1 (industry) or Step 2 (country), standard terminals force them into an unacceptable dilemma:
1. Press `Ctrl+C` to abort, throwing away all previously typed configuration and re-entering everything from scratch.
2. Complete the wizard with bad parameters and endure an unwanted or wasted scraping run.

### B. The PPA Stepper Standard
All interactive setup wizards across PPA Hubs and Spokes must implement a **Non-Destructive Bidirectional Stepper**:
1. **The Sentinel Return (`"__BACK__"`)**:
   - Interactive widgets (multi-select checklists, arrow menus, radio groups) accept `allow_back: bool = True`.
   - Keybindings bind `'b'`, `'back'`, and the `left` arrow key to immediately exit the prompt returning `"__BACK__"`.
   - Footer hints must display `[b] Back` alongside navigation keys.
   ```python
   if allow_back:
       @kb.add("b")
       @kb.add("left")
       def _step_back(event):
           event.app.exit(result="__BACK__")
   ```
2. **The State Machine Stepper Loop**:
   - Wizards must NEVER be structured as a cascading procedural sequence (`step1() -> step2() -> step3()`).
   - Wizards must be structured as a `while 1 <= step <= N:` state machine with an accumulated configuration dictionary:
   ```python
   step = 1
   state = {
       "industries": None,
       "countries": None,
       "regions": None,
       "platforms": None,
       "settings": None,
   }

   while 1 <= step <= 6:
       if step == 1:
           res = prompt_industry_selection(previous=state["industries"])
           if res is None:
               return None  # Cancelled
           state["industries"] = res
           step += 1

       elif step == 2:
           res = prompt_country_selection(previous=state["countries"])
           if res == "__BACK__":
               step = 1  # Step backward to Industry selection
               continue
           if res is None:
               return None
           state["countries"] = res
           step += 1
   ```
3. **Hierarchical / Intra-Loop Stepping**:
   - When configuring collections of sub-items (e.g. country-by-country state selection across 3 target countries), navigation operates hierarchically:
     - On country $i > 0$: pressing `b` steps backward to country $i - 1$, preserving country $i$'s selections.
     - On country $i = 0$: pressing `b` exits the sub-loop and returns to Step 2 (Country Selection).
4. **State Retention on Step-Back**:
   - When stepping backward to a previous step, all previously selected options and inputs are passed in as `previous_selection`.
   - Checkboxes, radio selections, or text inputs remain pre-filled with the user's prior choices, requiring zero re-typing unless the user explicitly wants to modify a field.
5. **Pre-Run Confirmation & Review Card**:
   - The final step (Step 6) renders a rich inspection panel summarizing all resolved parameters.
   - The action prompt provides 3 distinct paths: `[Y]es to run / [b]ack to settings / [c]ancel`. Pressing `b` steps backward to Step 5, while `c` cancels cleanly.

---

## 12. Virtual Console & Windows Screen Buffer Resilience Pattern

### A. The Core Architectural Defect
In Python `prompt_toolkit` applications running on Windows, calling `PromptSession()`, `prompt()`, or `Application()` without an explicit `output=` parameter triggers `prompt_toolkit.output.defaults.create_output()`.
On Windows:
1. `create_output()` tests if VT100 is supported via `is_win_vt100_enabled()`.
2. In IDE integrated terminals (VS Code, Cursor, Antigravity IDE), mintty, Git Bash, piped subprocesses, or pseudo-consoles, `GetConsoleMode()` or `SetConsoleMode()` fails.
3. `create_output()` falls back to `Win32Output(stdout)`.
4. `Win32Output.__init__()` invokes `GetConsoleScreenBufferInfo()`. Because the stream is redirected or lacks a direct Win32 `CONOUT$` buffer, it raises:
   ```text
   prompt_toolkit.output.win32.NoConsoleScreenBufferError: No Windows console found. Are you running cmd.exe?
   ```
5. Because this error occurs during startup, the application crashes immediately with `exit code 1`. If `cleanup_terminal_on_exit()` is invoked, it wipes the terminal via `os.system("cls")`, resulting in a silent failure with **zero visible error output**.

### B. The PPA Resilient Output Standard
Every PPA Hub and Spoke presentation layer (`ui.py` or entry module) must enforce **Virtual Console & Screen Buffer Resilience**:

```python
# ==============================================================================
# PROMPT_TOOLKIT RESILIENT OUTPUT INITIALIZATION (WINDOWS / VIRTUAL TERMINALS)
# ==============================================================================
import prompt_toolkit.output.defaults
from prompt_toolkit.output.vt100 import Vt100_Output
from prompt_toolkit.output.plain_text import PlainTextOutput

_original_create_output = prompt_toolkit.output.defaults.create_output

def create_robust_output(*args: Any, **kwargs: Any) -> Any:
    """Returns a prompt_toolkit Output instance that falls back gracefully on Windows
    when running without a standard Win32 console screen buffer (e.g. IDE terminals, pipes, mintty)."""
    try:
        return _original_create_output(*args, **kwargs)
    except Exception:
        try:
            return Vt100_Output.from_pty(sys.stdout)
        except Exception:
            return PlainTextOutput(sys.stdout)

# Monkey-patch prompt_toolkit's default output creator so all sessions and prompts
# in this process use robust fallback when no Win32 console buffer is available.
prompt_toolkit.output.defaults.create_output = create_robust_output
```

### C. Explicit Fallback Injection Mandate
In addition to the startup monkey-patch:
1. **`PromptSession`**: Always pass `output=create_robust_output()` during initialization.
2. **`Application`**: In full-screen or inline applications (e.g. `open_history_viewer`, `select_with_arrows`), always pass `output=create_robust_output()`.
3. **Subprocess Resilience**: Guarantees that invoking tools headlessly or via automation (`python -m <app>_cli --skill`) never raises console buffer exceptions in CI/CD or agent runners.


