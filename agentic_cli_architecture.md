# The Agentic CLI Architecture Blueprint 🤖🌐
> **Architectural Specification**: Complete engineering blueprint for transforming a modular plugin hub into an **AI Out-Of-The-Box Agentic CLI**, with OpenCode UI parity, multi-turn session memory, interactive session management (`/resume`, `/clear`, `/new`, `/delete`), visual transcript replay, stateless OpenRouter tool calling, interactive feature drilldown, and strict security isolation.

> **PPA Framework**: Hub layer — the agentic extension of the Universal CLI/TUI standard.
> **Upstream**: [CLI/TUI Hub Architecture](cli_architecture.md) — this document extends its visual & command standard · [Plugin Project Architecture (Root)](plugin_project_architecture.md)
> **Related**: [Bulk Scraper Engine Specification](bulk_scraper_engine_specification.md) — `/batch`, `/resume-batch` and Pattern 13 (schema-driven tool dispatch) · [Plugin SDK Specification](plugin_sdk_specification.md) — Hub tool schemas mirror each plugin's `SKILL.md` · [agent.md](agent.md) — framework map

---

## 1. Executive Summary & Paradigm Shift

Modern developer CLIs are transitioning from rigid, single-command utility scripts into **hybrid Agentic Hubs**. An Agentic CLI combines:
1. **Natural Language Understanding (NLU)**: Users can speak naturally (*"scrape the comments and download the latest mkbhd video"*).
2. **Multi-Turn Session Memory**: Context persists throughout the conversation session, allowing natural follow-up questions (*"is it done?", "summarize the transcript"*).
3. **Session Lifecycle Management**: First-class support for creating fresh sessions (`/new`), clearing history (`/clear`), browsing and resuming past sessions with visual replay (`/resume`), and purging sessions (`/delete`).
4. **Stateless Tool Calling**: LLM reasoning maps queries directly to structured CLI functions.
5. **Subprocess Fault Isolation**: Sub-tools execute in sandboxed child processes.
6. **Interactive TUI Parity**: Rich, stateful visual terminal interfaces matching modern standards (e.g., OpenCode).
7. **Dual-Agent Topology**: First-class support for both *Internal Agents* (the embedded OpenRouter AI in the TUI) and *External Coding Agents* (Codex, Claude Code, Antigravity) that discover and drive tools via the root `SKILL.md` and `--skill` CLI flags.

```text
                                  ┌────────────────────────────────┐
                                  │    User Input (REPL Prompt)    │
                                  └───────────────┬────────────────┘
                                                  │
                         ┌────────────────────────┴────────────────────────┐
                         ▼                                                 ▼
             ┌───────────────────────┐                         ┌───────────────────────┐
             │   Natural Language    │                         │  Direct Slash Command │
             │  (e.g., "get video")  │                         │   (e.g., "/youtube")  │
             └───────────┬───────────┘                         └───────────┬───────────┘
                         │                                                 │
                         ▼                                                 │
             ┌───────────────────────┐                                     │
             │   OpenRouter Agent    │                                     │
             │ (Multi-Turn + Memory) │                                     │
             └───────────┬───────────┘                                     │
                         │                                                 │
                         ▼                                                 ▼
             ┌─────────────────────────────────────────────────────────────────┐
             │               Subprocess Dispatcher (shell=False)               │
             │ ─────────────────────────────────────────────────────────────── │
             │  • `plugins/youtube-scrape/cli.py`                              │
             │  • `plugins/tiktok-scrape/cli.py`                               │
             └─────────────────────────────────────────────────────────────────┘
```

---

## 2. Dual-Mode State Topology (`AGENT_MODE` vs `MANUAL_MODE`)

A robust Agentic CLI must support two operating modes that can be switched on the fly:

| Attribute | **Agent Mode** (`/agent`) | **Manual Mode** (`/manual`) |
| :--- | :--- | :--- |
| **Default State** | Active by default on CLI launch | Secondary mode for scriptable workflows |
| **Prompt Glyph** | `  ┃ 🤖 > ` (Accent Blue `#5c9cf5`) | `  ┃ 👤 > ` (Warm Peach `#fab283`) |
| **Routing Layer** | OpenRouter AI Stateful LLM Agent | Direct Python Slash Command Router |
| **Active Tooltip** | `enter send    OpenRouter <model>` | `enter send    Manual Mode (/agent to switch)` |
| **Primary Use** | Fuzzy search, conversational queries, multi-turn follow-ups | Fast single-command execution, manual flag tweaking |

### Mode Toggle Implementation Pattern
```python
AGENT_MODE: bool = True  # Default = AI Agent Mode

def handle_agent_command():
    global AGENT_MODE
    AGENT_MODE = True
    render_action_card("[bold #5c9cf5]🤖 Agent Mode Activated[/bold #5c9cf5]")

def handle_manual_command():
    global AGENT_MODE
    AGENT_MODE = False
    render_action_card("[bold yellow]⌨️ Manual Mode Activated[/bold yellow]")
```

---

## 3. The OpenCode Visual Design Standard

### A. 3-Row Unicode Block Glyph Banner
Handcrafted unicode block glyphs provide crisp visual contrast without messy ASCII generator artifacts:
```python
# Muted Slate (#808080) for prefix + Crisp White (#eeeeee) for suffix
SOCIAL_LEFT = [
    "█▀▀█ █▀▀█ █▀▀▀ █ █▀▀█ █   ",
    "▀▀▀▄ █  █ █    █ █▀▀█ █   ",
    "█▄▄█ ▀▀▀▀ ▀▀▀▀ ▀ ▀  ▀ ▀▀▀▀",
]

SCRAPE_RIGHT = [
    "█▀▀█ █▀▀▀ █▀▀▄ █▀▀█ █▀▀█ █▀▀█",
    "▀▀▀▄ █    █▀▀▄ █▀▀█ █▀▀▀ █▀▀ ",
    "█▄▄█ ▀▀▀▀ ▀  ▀ ▀  ▀ ▀    ▀▀▀▀",
]
```

### B. Color Palette System
- **Primary Text (`#eeeeee`)**: Headers, active input text.
- **Muted Gray (`#808080`)**: Secondary metadata, timestamps, shortcut keys.
- **Border Neutral (`#3c3c3c`)**: Subtle card borders (`┃`), modal outlines.
- **Agent Blue (`#5c9cf5`)**: Agent badges, robot glyphs, tool routing notices.
- **Command Peach (`#fab283`)**: Model names, manual mode prompts.
- **Status Mint (`#73daca`)**: Active model indicators, health badges.

## 3. The OpenCode Visual Design Standard (Containerized TUI Paradigm)

Modern Agentic CLIs follow the **OpenCode Containerized TUI Architecture**, where the interface functions as a structured application container rather than an uncontrolled stream of raw terminal prints.

### A. Core Layout Anatomy
1. **Centralized Container Box**: The entire CLI sits inside a bounded container (max-width: 80–100 columns) centered horizontally in the terminal viewport.
2. **Internal Scrollable History**: Message cards and responses scroll *inside* the container, keeping the outer terminal background clean and static.
3. **Boxed Input Field**: The user prompt is enclosed in a dedicated bordered box at the bottom of the container.
4. **Drop-Down Autocompletion**: Slash command autocompletions render as a **Drop-Down** list descending directly below the input field.
5. **Floating Z-Index Modals**: Commands like `/plugins`, `/models`, or `/resume` open floating modal dialogs layered over the container.

### B. Logo & Branding Customization
The structural layout remains 1:1 identical to OpenCode across all projects, while the branding elements adapt dynamically:
- **3-Row Unicode Block Glyphs**: Use handcrafted unicode block glyphs (`█`, `▀`, `▄`) or compact ASCII banners (< 50 chars wide) to avoid horizontal wrapping on window resize.
- **Dynamic Palette Binding**: Bind the container borders, badges, and accents to the project's native theme (e.g. Cyan `#00ffff` for GSCRAPE, Slate/Peach for SocialScrape).

### C. Conversational Card Anatomy
Inside the container, conversation turns are structured into distinct visual cards:

#### 1. User Message Card (2-Line Bordered Card)
```text
  ┃  scrape mkbhd latest video
  ┃  USER (05:40 AM)
```
```python
def render_user_message(text: str) -> None:
    timestamp = time.strftime("%I:%M %p")
    console.print()
    console.print(f"[{COLOR_BORDER}]  ┃[/{COLOR_BORDER}]  [bold {COLOR_TEXT}]{text}[/bold {COLOR_TEXT}]")
    console.print(f"[{COLOR_BORDER}]  ┃[/{COLOR_BORDER}]  [{COLOR_MUTED}]USER ({timestamp})[/{COLOR_MUTED}]")
    console.print()
```

#### 2. AI Assistant Response (Dynamic Width & Wrapped Left-Border)
```text
  ╭─ GSCRAPE AI
  │  I have extracted the business contacts from Google Maps.
  ╰───────────────────────────────────────── Cohere/north-mini-code:free 05:40 AM
```
```python
def render_ai_response(content: str, model_id: str = "") -> None:
    timestamp = time.strftime("%I:%M %p")
    model_tag = (model_id or ACTIVE_MODEL).split("/")[-1]
    term_width = shutil.get_terminal_size((80, 24)).columns
    content_width = max(min(term_width - 8, 90), 35)

    console.print()
    console.print(f"[{COLOR_BORDER}]  ╭─ [bold cyan]GSCRAPE AI[/bold cyan] [dim cyan]({model_tag})[/dim cyan][/{COLOR_BORDER}]")
    for raw_line in content.strip().splitlines():
        if not raw_line.strip():
            console.print(f"[{COLOR_BORDER}]  │[/{COLOR_BORDER}]")
            continue
        wrapped = textwrap.wrap(raw_line, width=content_width, replace_whitespace=False, drop_whitespace=False)
        for w in wrapped:
            console.print(f"[{COLOR_BORDER}]  │[/{COLOR_BORDER}]  [white]{w}[/white]")
    console.print(f"[{COLOR_BORDER}]  ╰───────────────────────────────────────── [dim cyan]{timestamp}[/dim cyan][/{COLOR_BORDER}]")
    console.print()
```

---

## 4. Terminal Responsiveness & Anti-Tearing Best Practices

To prevent terminal tearing, line duplication, and ghost wrapping during window resizing (especially on Windows ConPTY / Windows Terminal), implement the following mandatory safeguards:

### A. Docked `bottom_toolbar` (Never Use Inline `rprompt` for Status)
- **The Issue**: Inline `rprompt` in `prompt_toolkit` fills the line with spaces up to the terminal edge. When a user resizes the terminal narrower, ConPTY wraps those blank spaces across multiple rows, permanently breaking the scrollback buffer.
- **The Best Practice**: Use `bottom_toolbar` for status information and shortcuts. A bottom toolbar is rendered as a docked status line that is transient and **never** written to the scrollback history:
```python
def get_toolbar(self):
    if AGENT_MODE:
        return HTML(f" <b>[Agent Mode]</b> Model: <b>{ACTIVE_MODEL.split('/')[-1]}</b> | <i>enter: send</i> | <i>/help: commands</i>")
    else:
        return HTML(" <b>[Manual Mode]</b> Slash commands active | <i>/agent to switch</i> | <i>/help: commands</i>")

user_input = self.session.prompt(
    HTML("<ansicyan>  ┃ 🤖 > </ansicyan>"),
    bottom_toolbar=self.get_toolbar
).strip()
```

### B. Deterministic Prompt Line Erasing
- **The Issue**: Pressing Enter leaves the raw typed input on screen before `render_user_message` displays.
- **The Best Practice**: Immediately execute ANSI cursor line-clearing upon prompt return before rendering the formatted card:
```python
user_input = self.session.prompt(...).strip()

# ANSI Line Erase: Erase raw input line and replace with rendered card
sys.stdout.write("\033[1A\033[2K\r")
sys.stdout.flush()

if user_input and not user_input.startswith("/"):
    render_user_message(user_input)
```

### C. Width-Aware Dynamic Paragraph Wrapping
- **The Issue**: Long lines from LLM responses wrap naturally at the terminal edge without the card border (`│`), causing text to spill outside the visual container.
- **The Best Practice**: Always wrap text with `textwrap.wrap(line, width=content_width)` before prefixing each wrapped sub-line with `  │  `.

### D. Compact ASCII Banner Bounds
- Keep ASCII art headers under 50 characters wide. Overly wide ASCII banners (> 70 columns) fracture and duplicate in the scrollback buffer when windows are resized.

### E. Terminal Viewport Reset, Screen Buffer & Mouse Scroll Architecture (OpenCode Standard)
- **The Design Intent**: Full-screen TUIs must balance the clean visual presentation of OpenCode (centered logo, shortcut grid, docked bottom toolbar) with **100% native terminal scrollability**.
- **The Root Cause of Scroll Lock & Input Hijacking**:
  1. **Alternate Screen Buffer (`\033[?1049h`)**: Entering the Alternate Screen Buffer disables the terminal emulator's native scrollback buffer. Lines that push off the top are permanently discarded, and the terminal's scrollbar is disabled.
  2. **Alternate Scroll Mode (`\033[?1007h`) & Mouse Tracking (`\033[?1000h` / `\033[?1006h`)**: When enabled, the terminal emulator converts mouse wheel movements into **Cursor Up and Down Arrow keys** (`\033[A` / `\033[B`). In `prompt_toolkit`, this causes mouse scrolling to cycle command history and type `/command` into the prompt input box instead of scrolling the terminal! Furthermore, terminal mode escape sequences are **sticky per terminal tab** and will persist across process restarts unless explicitly cleared.
  3. **How OpenCode Solves This (OpenCode Architecture Reference)**: OpenCode does not rely on terminal emulator scrollback. It uses `@opentui/core`'s virtual `ScrollBoxRenderable` within its SolidJS render tree with custom mouse wheel listeners (`useMouse: true`), managing scroll offsets programmatically inside the alternate screen canvas at 60 FPS.
- **The PPA Implementation Standard**:
  1. **Primary Buffer with Viewport Reset (`\033[2J\033[H`) by Default**: Instead of `\033[?1049h`, clear the visible viewport and home the cursor on launch. This hides the shell prompt (`PS C:\...>`) while keeping the terminal emulator's native scrollback memory 100% active.
  2. **Mandatory Hardware Mode Purge (`reset_terminal_mouse_mode`)**: Always execute an explicit escape sequence at module load, startup, screen clears (`/clear`), and exit to purge any sticky mouse interception modes:
     ```python
     TERMINAL_MOUSE_RESET = "\033[?1049l\033[?1007l\033[?1000l\033[?1002l\033[?1003l\033[?1006l\033[?1015l\033[?25h"

     def reset_terminal_mouse_mode() -> None:
         try:
             sys.stdout.write(TERMINAL_MOUSE_RESET)
             sys.stdout.flush()
         except Exception:
             pass
     ```
  3. **PromptSession Configuration (`mouse_support=False`)**: The main `PromptSession` must be initialized with `mouse_support=False` so the terminal emulator handles mouse wheel scrolling naturally.
  4. **Persistent Prompt History Navigation (`FileHistory`)**: Never use ephemeral `InMemoryHistory()`. Always initialize `PromptSession` with `FileHistory(str(data_dir / "history.txt"))` so users can press **Up Arrow** / **Down Arrow** at the prompt to scroll through previously typed commands across sessions.
  5. **In-App Scroll Viewer (`/history` & `PageUp` / `Ctrl+U`)**: Provide an interactive, scrollable chat history viewer (using a `prompt_toolkit.Application` with `full_screen=True` and `mouse_support=True`) allowing users to scroll smoothly through all previous turns, messages, and outputs using their mouse wheel, arrow keys, and PageUp/PageDown. Always call `reset_terminal_mouse_mode()` in its `finally:` block.
  6. **Opt-in Alternate Screen (`--alt-screen`)**: Support `--alt-screen` as an opt-in CLI flag for environments where strict screen isolation without scrollback is explicitly desired.

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
        # Initialize PromptSession with persistent history & in-app scroll keybindings
        history = FileHistory(str(get_data_dir() / "history.txt"))
        session = PromptSession(
            history=history,
            completer=completer,
            mouse_support=False,
            key_bindings=tui_keybindings, # PageUp & Ctrl+U bound to handle_history_command()
            bottom_toolbar=get_toolbar_html,
        )
        ...
    finally:
        reset_terminal_mouse_mode()
```

### F. Precise Multi-Line Banner Centering & Version Alignment
- **The Issue**: When centering a multi-line `Rich.Text` banner using `Align.center(banner)`, `Rich` computes centering based on the **longest line** in the entire object. If the version line has excessive leading/trailing spaces, the entire logo block gets pulled off-center to the left.
- **The Best Practice**: Calculate the exact width of the glyph rows dynamically, and set the version padding to exactly `total_width - len(version_string)`:
```python
def render_opencode_banner() -> Text:
    banner = Text()
    total_width = len(LOGO_LEFT[0]) + 1 + len(LOGO_RIGHT[0])
    
    for l, r in zip(LOGO_LEFT, LOGO_RIGHT):
        banner.append(l + " ", style=f"bold {COLOR_MUTED}")
        banner.append(r + "\n", style=f"bold {COLOR_PRIMARY}")

    ver_str = f"v{__version__}"
    # Exactly right-aligned flush under the right edge of the logo
    banner.append(" " * (total_width - len(ver_str)) + ver_str + "\n", style=f"{COLOR_MUTED}")
    return banner
```

### G. Uniform Intra-Word Character Spacing in 2-Tone Block Logos
- **The Standard**: When rendering a 2-tone block logo (e.g. Gray Prefix + Colored Suffix such as `G` gray + `SCRAPE` cyan or `OPEN` gray + `CODE` white), all letters within the word must maintain uniform 1-character spacing. The color transition between `l` and `r` should be separated by a single space (`l + " "` and `r`), preventing awkward gaps between split letters.

### H. Direct Filesystem Plugin Discovery & Collision-Free PluginLoader
- **The Standard**: Do not rely on `import plugins` or `pkgutil.iter_modules()`, which fail when:
  1. Multiple PPA tools are installed in the same virtual environment (e.g. `gscrape` and `socialscrape` both defining top-level `plugins`), causing namespace package collision.
  2. Plugin directories use PPA-standard kebab-case / hyphens (e.g. `gmaps-scraper`), which are invalid Python identifiers and skipped by `iter_modules()`.
- **The Proven Pattern**: Always scan `get_plugins_dir()` directly on the filesystem and dynamically load `core.py`, `__init__.py`, and `cli.py` using `importlib.util.spec_from_file_location`:
```python
class PluginLoader:
    @staticmethod
    def discover_plugins() -> Dict[str, PluginBase]:
        found_plugins: Dict[str, PluginBase] = {}
        plugins_dir = get_plugins_dir()
        if not plugins_dir.exists():
            return found_plugins

        # Ensure gscrape/plugins is prepended to sys.modules['plugins']
        if "plugins" not in sys.modules:
            mod_plugins = types.ModuleType("plugins")
            mod_plugins.__path__ = [str(plugins_dir)]
            sys.modules["plugins"] = mod_plugins
        elif hasattr(sys.modules["plugins"], "__path__"):
            if str(plugins_dir) not in sys.modules["plugins"].__path__:
                sys.modules["plugins"].__path__.insert(0, str(plugins_dir))

        for folder in sorted(plugins_dir.iterdir()):
            if not folder.is_dir() or folder.name.startswith((".", "_")):
                continue
            core_file = folder / "core.py"
            if not core_file.exists():
                continue

            norm_name = folder.name.replace("-", "_")
            init_file = folder / "__init__.py"

            # 1. Register package module for both hyphen and underscore aliases
            for pkg_name in [f"plugins.{folder.name}", f"plugins.{norm_name}"]:
                if pkg_name not in sys.modules:
                    spec = importlib.util.spec_from_file_location(
                        pkg_name, init_file if init_file.exists() else None, submodule_search_locations=[str(folder)]
                    )
                    if spec and spec.loader:
                        mod = importlib.util.module_from_spec(spec)
                        mod.__path__ = [str(folder)]
                        sys.modules[pkg_name] = mod
                        if init_file.exists():
                            spec.loader.exec_module(mod)

            # 2. Register core module
            core_spec = importlib.util.spec_from_file_location(f"plugins.{norm_name}.core", core_file)
            if core_spec and core_spec.loader:
                core_mod = importlib.util.module_from_spec(core_spec)
                sys.modules[f"plugins.{norm_name}.core"] = core_mod
                sys.modules[f"plugins.{folder.name}.core"] = core_mod
                core_spec.loader.exec_module(core_mod)

            # 3. Resolve PluginBase subclass and instantiate
            plugin_cls = getattr(core_mod, "__plugin_class__", None)
            if not plugin_cls:
                for attr_name in dir(core_mod):
                    attr = getattr(core_mod, attr_name)
                    if isinstance(attr, type) and issubclass(attr, PluginBase) and attr is not PluginBase:
                        plugin_cls = attr
                        break

            if plugin_cls and issubclass(plugin_cls, PluginBase):
                instance = plugin_cls()
                found_plugins[instance.name] = instance

        return found_plugins
```

## 5. Strict Slash Autocomplete Engine

### Design Constraints
1. **Slash-Triggered Exclusivity**: Autocomplete must **NEVER** appear during natural conversation (e.g., typing `"hello"`, `"download"`). It must trigger **ONLY** when the line starts with `/`.
2. **Dynamic Filtering**: Typing `/he` filters to `/help`. Typing a non-matching character (`/hexx`) cleanly hides the dropdown.
3. **Backspace Recovery**: Backspacing from `/hexx` to `/he` dynamically re-opens the dropdown. Backspacing past `/` cancels the completion menu.

### Implementation Pattern
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
        ("/update",   "pull latest updates from GitHub safely"),
        ("/fix",      "repair & reinstall Python dependencies"),
        ("/exit",     "terminate SocialScrape session"),
    ]

    def get_completions(self, document: Document, complete_event: CompleteEvent):
        text = document.text_before_cursor

        # 1. Strictly require leading slash
        if not text.startswith("/"):
            return

        # 2. Yield all on bare slash
        if text == "/":
            for cmd, desc in self.COMMANDS:
                yield Completion(cmd, start_position=-len(text), display=cmd, display_meta=desc)
            return

        # 3. Match prefix or substring
        query = text.lower()
        for cmd, desc in self.COMMANDS:
            if cmd.lower().startswith(query) or (len(query) > 1 and query[1:] in cmd.lower()):
                yield Completion(cmd, start_position=-len(text), display=cmd, display_meta=desc)

# Persistent Backspace Keybinding
tui_keybindings = KeyBindings()

@tui_keybindings.add("backspace")
def _(event):
    buf = event.current_buffer
    buf.delete_before_cursor(count=1)
    if buf.text.startswith("/"):
        buf.start_completion()
    else:
        buf.cancel_completion()
```

---

## 6. Interactive Feature Discovery & Zero Command Bloat

### The Anti-Pattern: Command Sprawl (`/features`, `/menu`, `/play`)
Creating standalone top-level slash commands for every minor sub-view leads to command bloat, cognitive overload, and messy help menus.

### The Best Practice: Nested Interactive Drilldown inside `/plugins`
Instead of having a duplicate `/features` command, the `/plugins` command serves as the primary resource inspector with live interactive drilldown:

```text
  📦 Active SocialScrape Plugins
  ┌────┬────────────────┬───────────────────┬──────────┬────────────────────────┐
  │ #  │ Plugin Name    │ Type              │ Status   │ Location               │
  ├────┼────────────────┼───────────────────┼──────────┼────────────────────────┤
  │ 1  │ youtube-scrape │ Python Plugin SDK │ ✅ Ready │ plugins/youtube-scrape │
  └────┴────────────────┴───────────────────┴──────────┴────────────────────────┘

  Type plugin number [1] to view full feature details (or Enter to return): 1
```

---

## 7. Session Memory Architecture & Chat Management Suite (`chats/`)

### A. The Stateless Follow-Up Problem & Tool Protocol
Without multi-turn session memory, every prompt is an isolated turn. If a user asks the agent to scrape a video and then follows up with *"Is it done?"*, the LLM receives only *"Is it done?"* without context and fails.
Furthermore, in OpenAI/Cohere/MiniMax protocol, an assistant message containing `tool_calls` **must** be immediately followed by a message with `"role": "tool"` and the exact matching `"tool_call_id"`.

### B. Session Storage Schema (`chats/<session_id>.json`)
```json
{
  "session_id": "session_20260827_060321",
  "title": "i want the transcript of this video...",
  "created_at": "2026-08-27 06:03:21",
  "active_model": "cohere/north-mini-code:free",
  "messages": [
    {
      "role": "user",
      "content": "i want the transcript of https://youtu.be/Itv0opM0Hug",
      "timestamp": "06:03 AM"
    },
    {
      "role": "assistant",
      "content": "",
      "tool_calls": [
        {
          "id": "call_12345",
          "type": "function",
          "function": {"name": "youtube_scrape", "arguments": "{\"url\": \"https://youtu.be/Itv0opM0Hug\"}"}
        }
      ],
      "timestamp": "06:03 AM"
    },
    {
      "role": "tool",
      "tool_call_id": "call_12345",
      "name": "youtube_scrape",
      "content": "Scraped YouTube video: 'Live Cold Calling' (ID: Itv0opM0Hug).\nTranscript Content Snippet:\nCould you do 3:15?...",
      "timestamp": "06:03 AM"
    }
  ]
}
```

### C. The 4 Session Management Handlers

#### 1. `/new` (Start Fresh Session)
```python
def handle_new_command():
    """Starts a fresh new chat session without needing to restart the terminal."""
    init_new_chat_session()
    display_home_screen()
    render_action_card(f"[bold green]✨ Started fresh chat session: {CURRENT_SESSION_ID}[/bold green]")
```

#### 2. `/clear` (Clean Active History)
```python
def handle_clear_command():
    """Clears all conversation messages for the current session."""
    global CONVERSATION_HISTORY
    CONVERSATION_HISTORY = []
    save_chat_session()
    display_home_screen()
    render_action_card("[dim]🧹 Cleared chat history for active session.[/dim]")
```

#### 3. `/resume` (Interactive Browser & Visual Transcript Replay)
```python
def handle_resume_command():
    """Renders interactive session picker and replays past turns visually into console."""
    global CURRENT_SESSION_ID, CONVERSATION_HISTORY
    session_files = sorted(CHATS_DIR.glob("*.json"), key=lambda p: p.stat().st_mtime, reverse=True)
    if not session_files:
        render_action_card("[bold yellow]⚠️ No saved chat sessions found.[/bold yellow]")
        return

    sessions_data = [(f, json.load(open(f, "r", encoding="utf-8"))) for f in session_files]
    table = Table(title="📂 Saved Chat Sessions (Select to Resume)", border_style=COLOR_BORDER)
    table.add_column("#", style="bold cyan", width=4)
    table.add_column("Session Title / First Query", style="bold white", width=38)
    table.add_column("Last Active", style="dim", width=19)
    table.add_column("Model", style="cyan", width=22)
    table.add_column("Turns", style="green", width=6)

    for idx, (f, data) in enumerate(sessions_data, 1):
        s_id = data.get("session_id", f.stem)
        title = data.get("title", "Empty Session")
        active_mark = " [bold green]● ACTIVE[/bold green]" if s_id == CURRENT_SESSION_ID else ""
        table.add_row(str(idx), f"{title[:35]}{active_mark}", data.get("created_at", ""), data.get("active_model", "").split("/")[-1], str(len(data.get("messages", []))))

    console.print()
    console.print(table)
    console.print()

    choice = input(f"Select session number [1-{len(sessions_data)}] to resume (or Enter to cancel): ").strip()
    if choice and choice.isdigit() and 1 <= int(choice) <= len(sessions_data):
        f, selected_data = sessions_data[int(choice) - 1]
        CURRENT_SESSION_ID = selected_data.get("session_id", f.stem)
        CONVERSATION_HISTORY = selected_data.get("messages", [])

        # Visual Replay
        display_home_screen()
        for msg in CONVERSATION_HISTORY:
            role, content = msg.get("role"), msg.get("content", "")
            if role == "user" and not content.startswith("[System Observation"):
                render_user_message(content)
            elif role == "assistant" and content:
                render_ai_response(content, selected_data.get("active_model", "SocialScrape AI"))
        render_action_card(f"[bold green]🔄 Resumed session: {CURRENT_SESSION_ID}[/bold green]")
```

#### 4. `/delete` (Interactive Deletion & Bulk Purge)
```python
def handle_delete_command(args: Optional[List[str]] = None):
    """Deletes single session or purges all saved sessions."""
    global CURRENT_SESSION_ID, CONVERSATION_HISTORY
    session_files = sorted(CHATS_DIR.glob("*.json"), key=lambda p: p.stat().st_mtime, reverse=True)
    if not session_files:
        render_action_card("[bold yellow]⚠️ No saved sessions to delete.[/bold yellow]")
        return

    if args and args[0].lower() in ("all", "--all", "-a"):
        for f in session_files: f.unlink()
        init_new_chat_session()
        render_action_card("[bold red]🗑️ All chat sessions deleted.[/bold red]")
        return
```

### D. Privacy & Data Isolation (`chats/` in `.gitignore`)
To protect user privacy and avoid committing transcripts to git:
```gitignore
# AI Chat Session Logs (Isolated per machine)
chats/*
!chats/.gitkeep
```

---

## 8. OpenRouter Orchestration & Strict Free Whitelist

### A. Why OpenRouter?
OpenRouter provides a unified OpenAI-compatible REST API across 400+ LLMs, allowing dynamic switching between Cohere, GLM, MiniMax, Poolside, and Nvidia without changing client logic.

### B. The Curated 8-Model Free Whitelist
Querying the raw OpenRouter model registry dynamically exposes experimental, paid, or deprecated endpoints (e.g. `google/gemini-2.0-flash-exp:free` returning HTTP 404). To ensure 100% uptime without surprise billing, strictly whitelist verified free models:

```python
DEFAULT_OPENROUTER_MODEL = "cohere/north-mini-code:free"

ALLOWED_OPENROUTER_MODELS = [
    {"id": "cohere/north-mini-code:free",           "name": "North Mini Code (free)",   "context_length": 256000},
    {"id": "z-ai/glm-5.2:free",                    "name": "GLM 5.2 (free)",           "context_length": 128000},
    {"id": "minimax/minimax-m3:free",              "name": "MiniMax-M3 Free",          "context_length": 1000000},
    {"id": "minimax/minimax-m2.7:free",            "name": "MiniMax-M2.7 (free)",      "context_length": 256000},
    {"id": "poolside/laguna-xs-2.1:free",          "name": "Laguna XS 2.1 (free)",     "context_length": 64000},
    {"id": "poolside/laguna-s-2.1:free",           "name": "Laguna S 2.1 (free)",      "context_length": 128000},
    {"id": "nvidia/nemotron-3-ultra-550b-a55b:free","name": "Nemotron 3 Ultra 550B",   "context_length": 128000},
    {"id": "nvidia/nemotron-3-super-120b-a12b:free","name": "Nemotron 3 Super 120B",   "context_length": 128000},
]
```

### C. Floating Model Picker Modal (`/models`)
Renders a clean numbered modal where typing `1`–`8` instantly switches and saves the active model to `.env`:
```text
  ╭─ Select Model ────────────────────────────────── esc ─╮
  │                                                       │
  │  Recent (Free Tier)                                   │
  │    [1] North Mini Code (free)            OpenRouter ● │
  │    [2] GLM 5.2 (free)                    OpenRouter   │
  │    [3] MiniMax-M3 Free                   OpenRouter   │
  │    ...                                                │
  ╰───────────────────────────────────────────────────────╯
```

---

## 9. Stateless Tool Calling & Subprocess Dispatching

### Tool Schema Definition
Plugin functions and introspection capabilities are registered as standard tool schemas:
```python
SOCIALSCRAPE_TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "youtube_scrape",
            "description": "Scrapes transcripts, metadata, comments, or downloads video/audio from YouTube URLs or channels.",
            "parameters": {
                "type": "object",
                "properties": {
                    "url": {"type": "string", "description": "YouTube video or stream URL"},
                    "channel": {"type": "string", "description": "YouTube channel handle"},
                    "download_video": {"type": "boolean", "description": "Download video file"},
                    "quality": {"type": "string", "enum": ["360p", "720p", "1080p", "4k", "best"]},
                    "extract_metadata": {"type": "boolean", "description": "Extract view count & description"},
                    "extract_comments": {"type": "boolean", "description": "Extract top comments"}
                }
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "view_features",
            "description": "Displays the complete feature matrix and subcommands of SocialScrape plugins (transcripts, metadata, comments, 4K video downloader, channel scrapers).",
            "parameters": {
                "type": "object",
                "properties": {
                    "plugin_name": {"type": "string", "description": "Optional plugin name to inspect (e.g. 'youtube-scrape')."}
                }
            }
        }
    }
]
```

### Dynamic Tool Dispatching & Schema-Driven Argument Builder ($O(1)$ Lookup)
Instead of hundreds of lines of repetitive `if/elif` statements, implement a declarative parameter mapping `TOOL_PARAM_FLAG_MAP` and universal argument parser `build_cmd_args()`:

```python
TOOL_PARAM_FLAG_MAP: Dict[str, Dict[str, Any]] = {
    "youtube_scrape": {
        "positional": ["url"],
        "flags": {
            "channel": "-c",
            "input_file": "--input-file",
            "type": "-t",
            "limit": "-l",
            "extract_metadata": "-m",
            "extract_comments": "--comments",
            "download_video": "--download-video",
            "download_thumbnail": "--download-thumb",
            "quality": "-q",
            "extract_all": "-a",
        },
        "display_name": "YouTube Scraper",
        "handler_key": "youtube",
    },
    "batch_scrape": {
        "positional": [],
        "flags": {
            "input_file": "--input-file",
            "download_video": "--download-video",
            "quality": "-q",
            "extract_all": "-a",
        },
        "display_name": "Batch Scraper Engine",
        "handler_key": "batch",
    },
}

def build_cmd_args(fn_name: str, fn_args: Dict[str, Any]) -> List[str]:
    """Dynamically maps JSON tool arguments into CLI flag tokens."""
    spec = TOOL_PARAM_FLAG_MAP.get(fn_name)
    if not spec:
        return []

    cmd_args: List[str] = []
    # 1. Positional arguments
    for pos_key in spec.get("positional", []):
        val = fn_args.get(pos_key)
        if val:
            cmd_args.append(str(val))

    # 2. Flag arguments
    for key, flag in spec.get("flags", {}).items():
        val = fn_args.get(key)
        if val is None or val is False:
            continue
        if isinstance(val, bool) and val is True:
            cmd_args.append(flag)
        else:
            cmd_args.extend([flag, str(val)])

    return cmd_args

def execute_tool_call(fn_name: str, fn_args: Dict[str, Any], handlers: Dict[str, Callable]) -> str:
    """Executes parsed tool call via O(1) dictionary dispatch."""
    spec = TOOL_PARAM_FLAG_MAP.get(fn_name)
    if not spec:
        return f"Unknown tool '{fn_name}'"

    cmd_args = build_cmd_args(fn_name, fn_args)
    handler = handlers.get(spec["handler_key"])
    if handler:
        handler(cmd_args)
    return f"{spec['display_name']} executed successfully."
```

---

## 10. Universal Command Architecture Matrix

| Command | Category | Description | Shortcut |
| :--- | :--- | :--- | :--- |
| `/agent` | Mode | Enable AI Agent Mode (natural language routing) | `ctrl+x a` |
| `/manual` | Mode | Enable Manual Mode (direct slash commands) | `ctrl+x m` |
| `/help` | Info | Display universal command reference table | `ctrl+x h` |
| `/batch` | Engine | Run bulk batch extraction from .txt/.csv input file | `-` |
| `/resume-batch` | Recovery | Resume an interrupted/incomplete batch from checkpoint | `-` |
| `/plugins` | Plugin | List active plugins with interactive feature drilldown | `ctrl+x p` |
| `/models` | Model | Browse & select active OpenRouter AI model | `ctrl+x o` |
| `/youtube` | Scraper | Download transcripts, metadata, comments & 4K video | `ctrl+x y` |
| `/instagram` | Scraper | Scrape profiles, reels, posts, comments & download MP4s | `ctrl+x i` |
| `/tiktok` | Scraper | Extract captions, feeds & watermark-free MP4s | `ctrl+x t` |
| `/linkedin` | Scraper | Scrape member profiles & company intelligence safely | `ctrl+x k` |
| `/twitter` | Scraper | Extract tweet threads, timelines & download MP4/GIFs | `ctrl+x w` |
| `/reddit` | Scraper | Extract subreddit discussions, comments & v.redd.it media | `-` |
| `/threads` | Scraper | Scrape Meta Threads user feeds & discussions | `-` |
| `/pinterest` | Scraper | Scrape pins, visual searches & download high-res media | `-` |
| `/meta` | Scraper | Search Meta Ad Library active campaigns & video ads | `-` |
| `/outputs` | Explorer | View all scraped folders and extracted media | `ctrl+x d` |
| `/new` | Session | Start a fresh new chat session (zero history) | `ctrl+x n` |
| `/history` | Session | Open interactive scrollable chat history viewer | `pageup` / `ctrl+u` |
| `/clear` | Session | Clear message history of current session & screen | `ctrl+l` |
| `/resume` | Session | Interactive browser to resume a saved session | `ctrl+x r` |
| `/delete` | Session | Delete saved session logs from disk | `-` |
| `/readme` | Doc | View full formatted project README documentation | `-` |
| `/skill` | Manifest | Render Master Hub SKILL.md or inspect a specific spoke plugin SKILL.md | `-` |
| `/skills` | Manifest | List all active agent skills, triggers, and CLI commands | `-` |
| `/config` | Config | View & edit global user preferences and configuration | `-` |
| `/fix` | Maintenance | Reinstall and repair all package dependencies | `-` |
| `/update` | Maintenance | Safely pull latest updates from remote repository | `-` |
| `/uninstall` | Maintenance | Display instructions to completely uninstall package | `-` |
| `/exit` | System | Gracefully terminate CLI session | `ctrl+c` |

---

## 11. Mistakes & Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails | Architectural Best Practice |
| :--- | :--- | :--- |
| **Missing `role: tool` following `tool_calls`** | OpenRouter/Cohere/MiniMax reject requests with HTTP 400 (`TOOL_CALLS_MISSING_RESULTS`). | Every assistant message with `tool_calls` must be followed by `role: "tool"` matching `tool_call_id`. |
| **Stateless Single-Turn API Calls** | LLM forgets past actions and cannot answer follow-up questions (*"is it done?"*). | Maintain multi-turn `CONVERSATION_HISTORY`, save to `chats/`, and feed tool observations into context. |
| **Alternate Screen / Sticky Mouse Trapping** | `\033[?1049h` disables terminal scrollback; `\033[?1007h` or sticky mouse tracking modes hijack wheel scrolls into Up/Down arrows (typing command history into prompt). | Use primary buffer with viewport reset (`\033[2J\033[H`) by default, set `mouse_support=False` on `PromptSession`, call `reset_terminal_mouse_mode()`, and provide `/history` scrollbox. |
| **Ephemeral `InMemoryHistory`** | Users cannot use Up/Down arrows to recall previous prompts after restart. | Always use `prompt_toolkit.history.FileHistory` pointing to `<data_dir>/history.txt`. |
| **Incomplete `pyproject.toml` `py-modules`** | Flat-layout Python modules not listed in `py-modules` are omitted from pip wheels/builds. | Explicitly list all root modules (`["socialscrape_cli", "cli", "storage", "ui", "agent", "batch_manager", "handlers"]`). |
| **Command Sprawl (`/features`, `/menu`)** | Clutters the root command namespace with single-use subviews. | Nest deep dives interactively within parent commands (`/plugins` -> select plugin). |
| **`shell=True` in Dispatcher** | Vulnerable to command injection via LLM hallucinated arguments. | Always use `shell=False` with explicit argument tokenization arrays `[*cmd_args]`. |
| **Heavy Vendor SDKs (`openai`)** | Bloats installation, creates version locking and slow startup times (>800ms). | Use lightweight native `requests` + `dotenv` (startup <100ms). |
| **Unchecked Model Registry** | 400+ models result in random 404s, rate-limits, and unexpected paid charges. | Curate a strict whitelist of verified free-tier models with fallback defaults. |
| **Uncontrolled Autocomplete** | Opening completions on regular words (`hi`, `test`) annoys users during chat. | Gate completer with `if not text.startswith('/'): return`. |
| **Alias Pollution & Underscore Commands in Completer** | Registering aliases (`/resumebatch`, `/resume_batch`, `/plugin`, `/chats`) and underscore commands in autocomplete produces confusing visual duplicates and breaks the PPA kebab-case naming standard. | Enforce strict two-word hyphenated kebab-case (`/resume-batch`). Register strictly ONE canonical command per feature/spoke in `_metadata`. Mark aliases with `is_alias=True` so they execute if typed but never pollute autocomplete dropdowns or `/help` tables. |
| **Raw Terminal Input Leftovers** | Leaving submitted input on screen results in duplicated visual text. | Use ANSI line clearing (`\033[1A\033[2K`) to replace input with rendered message cards. |

---

## 12. Architectural Summary & Blueprint Replication Guide

When replicating this architecture for any other modular CLI project:
1. **Root TUI Engine (`<tool>_cli.py`)**: Host the presentation layer, session memory, and OpenCode visual standard.
2. **Subprocess Spokes (`plugins/<plugin>/cli.py`)**: Keep each tool isolated, return structured data from `core.py`, and wrap with CLI argument parsing.
3. **Session Store (`chats/<session_id>.json`)**: Maintain session JSON files with auto-generated titles, timestamps, models, and multi-turn message arrays.
4. **Interactive Lifecycle Handlers**: Implement `/new`, `/history` (with mouse/keyboard scrolling), `/clear`, `/resume` (with visual replay), and `/delete`.
5. **Stateless Tool Gateway**: Map LLM tool calls directly to subprocess arguments using strict schema compliance.
