# PPA Bulk Resilient Scraping Engine Specification 🛡️⚡

> **Document Type**: Master Architectural Specification & Design Standard  
> **Version**: 2.0.0  
> **Target Audience**: Automation Engineers, Scraping Engine Architects, and AI Coding Agents (Claude, Codex, Antigravity)  
> **Location**: `.framework/PPA/bulk_scraper_engine_specification.md`

> **PPA Framework**: Hub layer — the batch & resilience engine (`batch_manager.py`) of the Root Hub.
> **Upstream (Root)**: [Plugin Project Architecture](plugin_project_architecture.md)
> **Related**: [Agentic CLI Architecture](agentic_cli_architecture.md) — Pattern 13 is its tool dispatch layer; this engine powers its `/batch` & `/resume-batch` commands · [CLI/TUI Hub Architecture](cli_architecture.md) — Pattern 15 defines the decoupled Hub file layout this engine lives in · [Plugin SDK Specification](plugin_sdk_specification.md) — Pattern 5 keeps spoke `core.py` decoupled from UI · [agent.md](agent.md) — framework map

---

## 1. Executive Summary & The "Bulk Resilience" Paradigm

Modern web scrapers and automation plugins in the **PPA (Plugin Project Architecture)** framework frequently process massive, multi-permutation search spaces (e.g., 50 regions $\times$ 30 target industries = 1,500 distinct queries, or bulk URL queues of 5,000+ items). 

A naive "Happy Path" loop (`for item in items: scrape(item)`) will inevitably fail in real-world production due to:
1. **Algorithmic Bot Fingerprinting**: Predictable, static execution intervals (`time.sleep(2)`) or linear scrolling trigger automated bot mitigation systems (Cloudflare, Google, DataDome, PerimeterX).
2. **Resource Exhaustion & Memory Leaking**: Long-running headless browser sessions (Chromium, Firefox, WebKit via Playwright/Selenium) accumulate leaked DOM trees, cache layers, and zombie processes, crashing after 100–300 pages.
3. **Cascading Failure on Network Blips**: A single DOM selector timeout or connection drop halts the entire multi-hour queue if not wrapped in self-healing isolation.
4. **Data Corruption on Interruptions**: Non-atomic writes corrupt output CSV/JSON files if the process is terminated mid-scrape.
5. **Lack of Checkpoint Recovery**: Unhandled user interruptions (`Ctrl+C`) or crashes force operators to restart entire bulk extractions from item #1.

```text
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                           PPA BULK RESILIENT SCRAPING ENGINE STANDARD                            │
├────────────────────────┬────────────────────────┬───────────────────────┬────────────────────────┤
│    STEALTH & PACING    │  LIFECYCLE & RETRIES   │  STATE & RECOVERY     │  DISPATCH & HARDENING  │
├────────────────────────┼────────────────────────┼───────────────────────┼────────────────────────┤
│ • Stochastic Jitter    │ • Proactive Recycling  │ • Real-Time JSON State│ • O(1) Tool Registry   │
│ • Macro-Batch Breaks   │ • Exponential Backoff  │ • /resume-batch Engine│ • build_cmd_args()     │
│ • Non-linear Scrolling │ • Subprocess Isolation │ • Multi-Format Parser │ • EAFP Fail-Fast Guards│
│ • Anti-Detect Cloaking │ • Timeout Enforcement  │ • Atomic File Writes  │ • Decoupled Telemetry  │
└────────────────────────┴────────────────────────┴───────────────────────┴────────────────────────┘
```

The **PPA Bulk Resilient Scraping Engine Standard** mandates 15 core architectural patterns to guarantee unattended reliability, zero IP bans, memory safety, 100% data integrity, and instant crash recovery across multi-thousand record runs.

---

## 2. The 15 Core Architectural Patterns

### Pattern 1: Stochastic Micro-Pacing (Human Jitter)
* **Principle**: Never execute actions at fixed time intervals. Emulate natural human variation with bounded stochastic jitter.
* **Rules**:
  - Pre-query delay: `random.uniform(2.0, 5.0)` seconds before launching a request or subprocess.
  - Scroll delay: `random.uniform(2000, 4000)` ms between dynamic infinite scroll evaluations.
  - Page load wait: `random.uniform(3000, 5500)` ms to let asynchronous AJAX requests settle.
* **Code Standard**:
```python
import random
import time

def apply_micro_delay(min_sec: float = 2.0, max_sec: float = 5.0) -> None:
    """Injects stochastic human-like delay between discrete actions."""
    delay = random.uniform(min_sec, max_sec)
    time.sleep(delay)
```

---

### Pattern 2: Macro-Batching & Stamina Cooldowns (Work Sessions)
* **Principle**: Emulate human work fatigue. Uninterrupted scraping across hundreds of requests flags behavioral heuristics on target domains.
* **Rules**:
  - Track `queries_since_last_macro_cooldown`.
  - Trigger a cooldown break every $N$ queries (default `15–20` queries).
  - Cooldown duration: `random.uniform(30.0, 60.0)` seconds (or 2–5 minutes for hyper-sensitive platforms).
  - Emit an explicit telemetry event (`☕ [Stealth Cooldown]`) to inform the user that the pause is deliberate.
* **Code Standard**:
```python
MACRO_BATCH_INTERVAL = 15
MACRO_COOLDOWN_MIN = 30.0
MACRO_COOLDOWN_MAX = 60.0

if queries_since_last_macro_cooldown >= MACRO_BATCH_INTERVAL:
    cooldown = random.uniform(MACRO_COOLDOWN_MIN, MACRO_COOLDOWN_MAX)
    emit("info", f"☕ [Stealth Cooldown] Pausing for {cooldown:.1f}s to emulate natural break...")
    time.sleep(cooldown)
    queries_since_last_macro_cooldown = 0
```

---

### Pattern 3: Proactive Resource & Browser Recycling
* **Principle**: Headless browser instances degrade over time. Proactively terminate and relaunch browser sessions before memory exhaustion or zombie processes occur.
* **Rules**:
  - Track `queries_since_last_browser_recycle`.
  - Recycle threshold: Every `25–30` queries.
  - Mechanism: Cleanly close the active page/context, allow garbage collection to reclaim memory, and instantiate a fresh session on the next iteration.
* **Code Standard**:
```python
BROWSER_RECYCLE_INTERVAL = 25

while position <= end_location:
    with browser_manager.launch(headless=headless) as browser:
        page = browser.new_page()
        while position <= end_location:
            if queries_since_last_browser_recycle >= BROWSER_RECYCLE_INTERVAL:
                emit("info", "🔄 [Stealth Maintenance] Recycling browser session to clear memory...")
                queries_since_last_browser_recycle = 0
                break  # Exits inner loop; outer loop instantiates fresh browser context
```

---

### Pattern 4: Isolated Per-Target Self-Healing Retry Loop (Browser Level)
* **Principle**: Individual target errors (network timeout, stale element reference, transient disconnection) must never terminate a bulk queue.
* **Rules**:
  - Wrap each discrete extraction attempt in an isolated retry loop with a maximum of `3` attempts (`MAX_QUERY_RETRIES = 3`).
  - If an attempt fails:
    1. Log a non-fatal warning event with attempt count (`⚠️ Query attempt 1/3 failed: ...`).
    2. Check if the page/browser is closed; if so, create a fresh page instance.
    3. Sleep 5 seconds before retrying.
  - If all 3 attempts fail: Record the failure in `run_manifest.csv`, mark the query as failed, and proceed seamlessly to the next item in the queue.

---

### Pattern 5: Decoupled Telemetry & Two-Tier Progress Architecture
* **Principle**: Pure SDK code (`core.py`) must remain 100% decoupled from console rendering. It must NEVER import `rich` or call `print()`. Progress tracking must be organized into a two-tier hierarchy: a high-level **Macro Batch Progress Bar** (for overall queues) and low-level **Micro-Step Telemetry** (for in-flight scraping operations).
* **Rules**:
  - `core.py` publishes signals through an event callback (`event_callback(event_type, msg)`) or central `EventBus`.
  - Signal categories:
    - `"info"`: Operational status (searching query, finished batch, breaks).
    - `"progress"`: Micro-step status (scrolled N cards, delay countdowns, subpage crawls).
    - `"warning"`: Recoverable retry notices or skipped duplicates.
    - `"error"`: Fatal query failures or parsing exceptions.
    - `"success"`: Saved batch files, master file writes, verified entity notices.
  - **Macro-Level Progress Bar (`rich.progress.Progress`)**:
    - The presentation layer (`cli.py` or `ui.py`) initializes a persistent `rich.progress.Progress` instance tracking high-level units of work (e.g., Target Locations, Input URLs, or Rows to Process).
    - Columns must include: `[SpinnerColumn, TextColumn, BarColumn, TaskProgressColumn, MofNCompleteColumn, TimeElapsedColumn]`.
  - **Micro-Level Event Streaming**:
    - Detailed operations (e.g., cards scraped `[13/54]`, digital verification, AI audits) stream to the terminal via `progress.console.print(...)` without overwriting or tearing the docked progress bar.
  - **Reliable Advance Contract (Anti-Stall Guarantee)**:
    - The presentation layer must advance the macro task whenever a macro unit is processed, regardless of whether internal queries had recoverable failures or partial errors.
    - Never couple progress bar advancement to 100% error-free completion; if a location finishes with partial errors, the macro task MUST still advance (`progress.advance(task_id)`), ensuring the bar never permanently freezes at `0%` during resilient multi-hour runs.
  - **Run Parameter Audit Standard (`run_logs/runs.csv`)**:
    - Before launching batch processing, the CLI or runner must log all configured parameters (timestamp, catalog, filters, countries, states, location counts, limits, mode) into `run_logs/runs.csv` via the shared `OutputManager.log_run_parameters()` standard. This enables full auditability, debugging, and cross-run parameter comparison.

---

### Pattern 6: Real-Time Incremental Row Persistence (Anti-Buffering Standard) & Atomic Writes
* **Principle**: Processing engines (scraping, enrichment, verification, and auditing) MUST NOT buffer processed records in memory waiting for entire queries, locations, or batches to conclude before persisting to disk. Every unit of extracted or enriched data must be written immediately. Mid-run terminations or crashes must never cause data loss for already processed records.
* **Rules**:
  - **Zero End-of-Batch Delay**: Scraper and processor loops must immediately persist each processed record to disk (both master CSVs and batch files) as soon as it is extracted or audited, rather than accumulating rows in an in-memory list to save only when the query or batch concludes.
  - **Immediate External Visibility**: Incremental row-by-row persistence ensures monitors, operators, downstream plugins, and spreadsheet viewers immediately observe incoming records in real time.
  - **Atomic File Writing**: When saving entire tables, indexes, or sync logs, write new data to a temporary file (`.tmp`) and atomically replace the target file via filesystem move operations.
  - **Deduplication Keys**: Compute deterministic hashes/keys (e.g. `record_key = make_record_key(place_url, name, phone, website)`) and check against `master_businesses.csv` in RAM using a Set (`O(1)` lookup) before appending.
* **Code Standard**:
```python
# Real-Time Incremental Row-by-Row Persistence
for lead in leads:
    record = enrich_lead(lead)
    # Persist immediately to disk - zero batch buffer delay
    output_manager.upsert_company(master_csv_path, record)
    append_row_to_csv(batch_csv_path, record)
    emit("progress", f"Saved {record['name']} to disk")

@staticmethod
def write_csv_atomic(output_file: Path, records: List[Dict[str, Any]], fieldnames: List[str]) -> None:
    """Atomically writes records using a temporary file replacement."""
    output_file.parent.mkdir(parents=True, exist_ok=True)
    temp_file = output_file.with_suffix(".tmp")
    with open(temp_file, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
        writer.writeheader()
        writer.writerows(records)
    temp_file.replace(output_file)
```

---

### Pattern 7: Dynamic Custom Target Injection (Ad-Hoc Mode)
* **Principle**: Predefined data registries (e.g. standard country lists or industry categories) must never restrict ad-hoc usage.
* **Rules**:
  - Inject `[Custom Input]` choices at the top index of TUI multi-select menus.
  - If selected, prompt the operator for comma-separated inputs.
  - Pass custom strings directly into runtime queues without mutating the underlying base CSV/JSON asset files.

---

### Pattern 8: Stealth Cloaking, Zero Paid API & Automated Consent Bypass
* **Principle**: Avoid costly commercial scraping APIs and uncloaked browser automation. Automated browser sessions must navigate seamlessly without getting blocked by anti-bot checks, IP bans, or interactive cookie consent dialogs.
* **Rules**:
  - Use fingerprint-cloaked browser engines (e.g., `Camoufox` built on anti-detect Firefox forks).
  - Emulate natural user-agent headers, viewport sizes, WebGL vendor strings, and TLS/JA3 fingerprints.
  - Enforce zero external API dependencies for scraping operations.
  - **Zero-Prerequisite Binary Resolution**: Scraper engines must verify local binary presence via `is_browser_installed()` and transparently auto-fetch missing binaries inside `BrowserManager.launch()` so operators never encounter missing-executable crashes.
  - **Automated Cookie & Consent Banner Dismissal (`dismiss_consent_banners`)**:
    - Unattended scrapers encounter modal cookie consent dialogs (e.g. Google Maps "Before you continue", GDPR notices, OneTrust, Cookiebot, TrustArc) that obscure interactive elements and cause locator timeouts.
    - Browser scraper engines must implement an automated, non-blocking consent dismissal protocol executed immediately after `page.goto()` and inside retry/wait loops.
    - The dismissal function must probe a multi-lingual selector ladder (e.g. `button:has-text('Accept all')`, `button:has-text('Tout accepter')`, `button:has-text('Alle akzeptieren')`, `button:has-text('Acepto')`, `button:has-text('I agree')`, `form[action*='consent'] button`, `button[aria-label*='Accept']`, `#L2AGLb`) with a short timeout (1–2 seconds) and wrapped in a fail-safe try/except block so page execution is never halted if no banner exists.

---

### Pattern 9: Universal Multi-Format Target Ingestion (`batch_manager.py`)
* **Principle**: Bulk engines must seamlessly ingest target lists from `.txt`, `.csv`, or `.md` files with zero manual reformatting.
* **Rules**:
  - Support `.txt` and `.md` line-by-line parsing, automatically extracting markdown links `[title](url)` and stripping inline comments (`#`).
  - Support `.csv` files by auto-detecting common column headers (`url`, `target`, `link`, `handle`, `username`, `keyword`, `id`) or falling back to index 0.
  - Deduplicate targets while strictly preserving input order.
* **Code Standard**:
```python
def load_targets_from_file(file_path: Path | str) -> List[str]:
    path = Path(file_path).resolve()
    if not path.exists():
        raise FileNotFoundError(f"Input file not found: {path}")

    targets: List[str] = []
    seen = set()
    suffix = path.suffix.lower()

    if suffix == ".csv":
        with open(path, "r", encoding="utf-8", errors="replace") as f:
            reader = csv.reader(f)
            header_checked = False
            col_idx = 0
            for row in reader:
                if not row:
                    continue
                if not header_checked:
                    headers = [c.strip().lower() for c in row]
                    for idx, h in enumerate(headers):
                        if h in ("url", "target", "link", "handle", "username", "id", "keyword"):
                            col_idx = idx
                            break
                    header_checked = True
                    if any(h in ("url", "target", "link", "handle") for h in headers):
                        continue
                if len(row) > col_idx:
                    val = row[col_idx].strip()
                    if val and val not in seen and not val.startswith("#"):
                        seen.add(val)
                        targets.append(val)
    else:
        with open(path, "r", encoding="utf-8", errors="replace") as f:
            for line in f:
                clean = line.strip()
                if not clean or clean.startswith("#"):
                    continue
                # Extract markdown links [Title](URL)
                md_links = re.findall(r"\[.*?\]\((https?://[^\s\)]+)\)", clean)
                if md_links:
                    for link in md_links:
                        if link not in seen:
                            seen.add(link)
                            targets.append(link)
                    continue
                # Extract plain URLs or raw target strings
                urls = re.findall(r"(https?://[^\s\)\*\<\>\"']+)", clean)
                if urls:
                    for u in urls:
                        if u not in seen:
                            seen.add(u)
                            targets.append(u)
                elif clean not in seen:
                    seen.add(clean)
                    targets.append(clean)

    return targets
```

---

### Pattern 10: O(1) Append-Only Checkpointing (`batches/<batch_id>.jsonl`)
* **Principle**: State must be saved in real time to prevent data loss, but writing large JSON lists iteratively causes $O(N^2)$ IO bottlenecks on hyper-scale lists (100,000+ items). The solution is a **JSON Lines (JSONL) Event Log**.
* **Rules**:
  - Assign every batch a unique timestamped ID (`batch_YYYYMMDD_HHMMSS`).
  - Do not rewrite the file. Instead, *append* single-line JSON events in $O(1)$ time whenever state changes.
  - Reconstruct state on resumption by reading the `.jsonl` file top-to-bottom to replay events.
  - State file schema (`.jsonl` format):
```jsonl
{"event": "batch_init", "timestamp": 1690000000.0, "batch_id": "batch_2026", "platform": "youtube", "total_items": 100}
{"event": "item_update", "timestamp": 1690000001.0, "target": "https://youtube.com/watch?v=123", "status": "pending"}
{"event": "item_update", "timestamp": 1690000002.0, "target": "https://youtube.com/watch?v=123", "status": "completed", "duration_seconds": 1.0, "retries": 0}
{"event": "batch_status", "timestamp": 1690000003.0, "status": "completed"}
```

---

### Pattern 11: Exponential Backoff Subprocess Retry Loops
* **Principle**: Transient network drops and rate limits must be retried with exponentially increasing backoff before being recorded as fatal.
* **Rules**:
  - Enforce subprocess execution timeout (default: `180s`) to terminate hanging network requests.
  - Wrap subprocess execution in `for attempt in range(max_retries):` (default `max_retries = 3`).
  - On failure or timeout: Calculate backoff `(2 ** attempt) + random.uniform(0.5, 1.5)` seconds.
* **Code Standard**:
```python
MAX_RETRIES = 3
success = False

for attempt in range(MAX_RETRIES):
    item_start = time.time()
    try:
        res = subprocess.run(cmd, cwd=ROOT_DIR, timeout=180)
        item.duration_seconds += (time.time() - item_start)
        if res.returncode == 0:
            item.status = "completed"
            item.error = None
            success = True
            break
        else:
            item.retries += 1
            item.error = f"Exit code {res.returncode}"
            if attempt < MAX_RETRIES - 1:
                backoff = (2 ** attempt) + random.uniform(0.5, 1.5)
                time.sleep(backoff)
    except subprocess.TimeoutExpired:
        item.duration_seconds += (time.time() - item_start)
        item.retries += 1
        item.error = "Process timed out (180s)"
        if attempt < MAX_RETRIES - 1:
            backoff = (2 ** attempt) + random.uniform(1.0, 2.0)
            time.sleep(backoff)

if not success:
    item.status = "failed"
```

---

### Pattern 12: Zero-Data-Loss Resumption Mechanism (`/resume-batch`)
* **Principle**: Resuming an interrupted batch must require a single command and zero manual configuration.
* **Rules**:
  - Provide a dedicated `/resume-batch` slash command and CLI argument (`socialscrape batch --resume`).
  - On TUI startup, proactively scan `batches/` for incomplete batch states (`status != "completed"` or `pending_count > 0`). If detected, render a prominent warning card alerting the user.
  - When resumed, load the existing checkpoint, filter out all items marked `"completed"`, execute remaining items, and update the existing checkpoint file in place.
* **Code Standard**:
```python
def get_incomplete_batches() -> List[Dict[str, Any]]:
    incomplete = []
    for f in sorted(BATCHES_DIR.glob("*.json"), key=lambda p: p.stat().st_mtime, reverse=True):
        try:
            with open(f, "r", encoding="utf-8") as fp:
                data = json.load(fp)
                if data.get("status") != "completed" or data.get("pending_count", 0) > 0:
                    incomplete.append(data)
        except (json.JSONDecodeError, IOError, OSError):
            continue
    return incomplete
```

---

### Pattern 13: Schema-Driven Universal Tool Dispatching ($O(1)$ Lookup)
* **Principle**: Replace giant monolithic `if/elif` chains in LLM agent tool dispatchers with a declarative schema mapping and dictionary dispatch.
* **Rules**:
  - Define `TOOL_PARAM_FLAG_MAP` mapping JSON schema keys to CLI flags (`positional` vs `flags`).
  - Implement a universal `build_cmd_args(fn_name, fn_args)` helper.
  - Register command handlers in a `TOOL_HANDLERS` dictionary for $O(1)$ dispatch execution.
* **Code Standard**:
```python
TOOL_PARAM_FLAG_MAP = {
    "youtube_scrape": {
        "positional": ["url"],
        "flags": {
            "channel": "-c",
            "input_file": "--input-file",
            "extract_comments": "--comments",
            "download_video": "--download-video",
            "quality": "-q",
            "extract_all": "-a",
        },
        "handler_key": "youtube",
    }
}

def build_cmd_args(fn_name: str, fn_args: Dict[str, Any]) -> List[str]:
    spec = TOOL_PARAM_FLAG_MAP.get(fn_name, {})
    cmd_args = []
    for pos in spec.get("positional", []):
        if fn_args.get(pos):
            cmd_args.append(str(fn_args[pos]))
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
    spec = TOOL_PARAM_FLAG_MAP.get(fn_name)
    if not spec:
        return f"Unknown tool '{fn_name}'"
    cmd_args = build_cmd_args(fn_name, fn_args)
    handler = handlers.get(spec["handler_key"])
    if handler:
        handler(cmd_args)
    return f"Tool {fn_name} executed successfully."
```

---

### Pattern 14: EAFP Fail-Fast & Explicit UI Error Hardening
* **Principle**: Never swallow errors blindly via `except Exception: pass`. Catch narrow, explicit exception types and emit actionable UI notifications.
* **Rules**:
  - Always specify exact exception types: `(json.JSONDecodeError, IOError, OSError, PermissionError, subprocess.SubprocessError)`.
  - In TUI presentation layers, display transient error warning cards (`render_action_card(...)`) so the operator is immediately aware of disk, permission, or network issues without crashing the REPL loop.

---

### Pattern 15: Layered Decoupled Hub Architecture Standard
* **Principle**: The Root Hub must be strictly split into 6 decoupled architectural layers to maintain modularity, readability, and testability.
* **Standard Hub Layout**:
```text
<app_root>/
├── ui.py                    # Presentation Layer (Rich themes, OpenCode banners, cards, docked toolbar)
├── agent.py                 # AI Core (OpenRouter model whitelist, multi-turn memory, stateless tool schemas)
├── handlers.py              # Command & Subprocess Layer (Sandboxed subprocess runners, modal views)
├── batch_manager.py         # Batch & Resilience Engine (State persistence, retries, jitter, resumption)
├── storage.py               # Path & Storage Engine (Dual-mode global ~/.<app> vs local development)
├── <app>_cli.py             # Master TUI Controller (Terminal lifecycle, autocomplete, REPL loop)
├── cli.py                   # Root shortcut runner
├── batches/                 # Fail-safe batch state checkpoints (.json)
└── chats/                   # Multi-turn conversation session logs (.json)
```

---

### Pattern 16: Zero-Tradeoff Scraper Performance Audit Protocol
* **Principle**: Before optimizing any scraping or batch extraction plugin (e.g. `gmaps-scraper`, `social-deepscrape`), engineers must perform an isolated phase latency audit to isolate true bottlenecks without degrading data output quality, stealth, or code health.
* **Rules**:
  1. **Discrete Phase Profiling**: Measure runtime across the 4 distinct lifecycle stages:
     - Stage 1: *Primary Directory Search / Listing Harvest* (e.g. Google Maps, LinkedIn search results).
     - Stage 2: *Deep Target Ingestion* (Crawling company websites, subpages `/contact`, `/about`).
     - Stage 3: *External Verification & Dorking* (Social checks, MX lookups, search engine queries).
     - Stage 4: *Audit Scoring & Serialization* (DOM inspections, tech stack scoring, CSV writes).
  2. **Waste Ratio Calculation**: Identify where idle network time and redundant DOM visits occur. If >70% of runtime is spent waiting on third-party websites while directory listing is <15%, optimizations must target the enrichment workers, not the directory listing.
  3. **Zero-Tradeoff Mandate**: Optimizations must strictly preserve:
     - 100% of existing extracted data fields.
     - Full anti-bot stealth and fingerprint evasion.
     - Strict MVC decoupling (zero `print()` in `core.py`).
     - Zero root directory pollution (`Outputs/` and `run_logs/` remain in isolated spoke folders).

---

### Pattern 17: High-Throughput `BrowserPool` Multi-Context Concurrency Standard
* **Principle**: Avoid spawning multiple heavy browser OS processes. Scale concurrency using lightweight, isolated `BrowserContext` instances within a shared browser instance, combined with resource route aborting.
* **Rules**:
  1. **Multi-Context Isolation**: Use `browser.new_context()` to isolate cookies, cache, local storage, and fingerprinting state per worker. Each context consumes only ~25–40MB RAM (compared to ~300MB+ for a separate browser process), allowing 4 to 8 parallel workers to run under ~300MB of RAM total.
  2. **Domain-Aware Concurrency Rule**:
     - *Primary Rate-Limited Directories (e.g. Google Maps)*: Strictly single-stream or human-jittered pacing (2–5s) to avoid triggering CAPTCHAs or IP bans.
     - *Disparate Target Domains (e.g. small business websites)*: 4 to 8 parallel consumer workers concurrently. Because each worker queries a different domain across the internet, there is zero risk of domain IP rate-limiting.
  3. **Aggressive Media & Asset Aborting**: Intercept and abort non-essential binary assets to reduce page load time from ~3.5s down to ~0.4s and save 85% bandwidth:
     ```python
     context.route("**/*.{png,jpg,jpeg,webp,svg,woff,woff2,mp4,gif,ico,css}", lambda r: r.abort())
     ```
* **Code Standard**:
```python
class CamoufoxWorkerPool:
    """Manages high-throughput parallel BrowserContext instances with zero memory leaks."""
    def __init__(self, max_workers: int = 5):
        self.max_workers = max_workers
        self.browser = None

    def start(self, headless: bool = True):
        self.browser = Camoufox(headless=headless)

    def create_isolated_context(self):
        context = self.browser.new_context()
        # Abort heavy binary media automatically
        context.route("**/*.{png,jpg,jpeg,webp,svg,woff,woff2,mp4,gif,ico}", lambda r: r.abort())
        return context

    def close(self):
        if self.browser:
            self.browser.close()
```

---

### Pattern 18: Fast-Path HTTP Ingestion & Pipeline Coalescing
* **Principle**: Do not launch a heavy browser engine when static HTML is sufficient, and never visit the exact same website multiple times across separate plugin stages.
* **Rules**:
  1. **Fast-Path HTTP First (90% Bypass)**:
     - Small business websites (WordPress, Squarespace, static HTML) include emails, phone numbers, and social links directly in the initial HTML payload.
  - Query target sites via a fast, TLS-cloaked HTTP client (`curl_cffi` or `httpx`). Parse raw HTML in ~50ms.
     - Fall back to a Camoufox browser context *only* if the response is an empty JavaScript SPA (React/Vue) or Cloudflare challenge.
  2. **Single-Pass Ingestion (Pipeline Coalescing)**:
     - When multiple plugins or steps need data from the same target URL (e.g. `social-deepscrape` for contact info and `lead-ai-auditor` for SEO/speed scoring), coalesce them into a **single visit**.
     - Extract contact links, audit load time, evaluate tech stack, and score CRO forms simultaneously from that one DOM and network response.
   3. **High-Concurrency Async Verification**:
      - Perform email MX DNS lookups and social profile HEAD requests asynchronously via `asyncio` and connection pooling, reducing verification from ~5s down to ~350ms per lead.

---

### Pattern 19: HTTP-First Search Dorking & Loop-Safe Automation Standard
* **Principle**: Automated search dorking (e.g., querying Bing, DuckDuckGo, Yahoo for executive decision makers or LinkedIn profiles) must prioritize lightweight HTTP requests over browser automation. Furthermore, synchronous browser automation APIs (such as Playwright sync) must **never** be invoked on an active asynchronous event-loop thread (e.g., `prompt_toolkit` or `asyncio` loops in interactive TUIs), which triggers uncatchable `Playwright Sync API inside asyncio loop` greenlet deadlocks and 10–12s hanging timeouts per query.
* **Rules**:
  1. **HTTP-First Search Queries**:
     - Search engines return result listings in static HTML without requiring full JavaScript rendering.
     - Dorking queries must be dispatched directly via an HTTP client (`FastPathHttpClient` / `httpx`) configured with desktop headers (`User-Agent`, `Accept-Language`, `sec-ch-ua`).
     - Response parsing (e.g., `BeautifulSoup`) takes < 200ms per engine compared to 8–15s for full browser navigation.
  2. **URL-Safe Base64 Redirect Unmasking**:
     - Search engines frequently mask outbound links behind tracking redirects (e.g., Bing `/ck/a?!&&p=...&u=a1<base64>&...`).
     - Decouple destination extraction from browser navigation: decode the URL-safe base64 string (`base64.urlsafe_b64decode`) directly in RAM to extract the true target link (e.g., LinkedIn profile) instantly without network or browser overhead.
  3. **Event-Loop & Thread-Affinity Safety (No Playwright Sync in Asyncio Loop)**:
     - If browser rendering is mandatory (e.g., complex SPAs or anti-bot JS challenges), Camoufox/Playwright sync instances MUST execute inside dedicated worker threads or isolated context pools (`browser_manager.context_pool(max_contexts=1)`).
     - Under no circumstances may synchronous browser `launch()` or `new_page()` be called from an async thread or prompt_toolkit REPL thread.

---

### Pattern 20: Configurable Multi-Channel Filtering & Unconditional Core Attribute Standard
* **Principle**: In deep contact enrichment pipelines, scrapers must allow operators to selectively filter secondary communication and social channels, while strictly guaranteeing that primary entity identity attributes remain unconditionally preserved.
* **Rules**:
  1. **Explicit Channel Taxonomy**:
     - All supported secondary channels must be registered in an immutable taxonomy constant (e.g. `SUPPORTED_SOCIAL_PLATFORMS = ("linkedin", "twitter", "instagram", "facebook", "tiktok", "youtube", "reddit")`).
  2. **Early-Exit Dorking & Crawl Bypass**:
     - When an operator filters platforms (via `--social-platforms` or interactive checklist), search engine dorking routines for excluded channels must be bypassed entirely.
     - Example: If LinkedIn is omitted, search queries targeting `site:linkedin.com/in/` are skipped, saving 1–3 seconds per lead and eliminating rate-limit exposure.
  3. **The Unconditional Core Attribute Law (Mandatory Negative Constraint)**:
     - Core business identity attributes (`website`, `phone_number`, `emails`, `address`, `decision_makers`) must **NEVER** be omitted or subjected to user channel filters.
     - Filtering applies strictly to secondary social channels and must be isolated to the unified single-column `social_links` attribute.
  4. **Single-Column Unified Serialization**:
     - Multi-channel results must serialize to a single unified string (e.g. `"linkedin: https://..., twitter: https://..."`) containing exclusively the whitelisted channels, eliminating column spillover and maintaining strict CSV schema locks.

---


## 3. Reference Implementation: Standard `batch_manager.py`

Below is the complete reference implementation of a PPA-compliant `batch_manager.py` engine:

```python
"""
PPA Standard Batch Manager Reference Engine (`batch_manager.py`).
"""

import csv
import json
import os
import random
import re
import subprocess
import sys
import time
from dataclasses import asdict, dataclass
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

@dataclass
class BatchItem:
    target: str
    status: str = "pending"  # "pending", "in_progress", "completed", "failed"
    error: Optional[str] = None
    duration_seconds: float = 0.0
    retries: int = 0

    def to_dict(self) -> Dict[str, Any]:
        return asdict(self)

    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> "BatchItem":
        return cls(**data)


class BatchQueue:
    def __init__(
        self,
        platform: str,
        cli_script: Path,
        root_dir: Path,
        state_dir: Path,
        targets: Optional[List[str]] = None,
        extra_args: Optional[List[str]] = None,
        delay_range: Tuple[float, float] = (2.0, 5.0),
        batch_id: Optional[str] = None,
        max_retries: int = 3,
    ):
        self.platform = platform
        self.cli_script = Path(cli_script)
        self.root_dir = Path(root_dir)
        self.state_dir = Path(state_dir)
        self.extra_args = extra_args or []
        self.delay_range = delay_range
        self.batch_id = batch_id or time.strftime("batch_%Y%m%d_%H%M%S")
        self.max_retries = max_retries
        self.status = "in_progress"
        self.state_file = self.state_dir / f"{self.batch_id}.jsonl"
        self.items: List[BatchItem] = [BatchItem(target=t) for t in (targets or [])]

    def _append_event(self, event_type: str, payload: Dict[str, Any]) -> None:
        self.state_dir.mkdir(parents=True, exist_ok=True)
        with open(self.state_file, "a", encoding="utf-8") as f:
            event = {"event": event_type, "timestamp": time.time()}
            event.update(payload)
            f.write(json.dumps(event, ensure_ascii=False) + "\n")

    def init_checkpoint(self) -> None:
        self._append_event("batch_init", {"batch_id": self.batch_id, "platform": self.platform, "total_items": len(self.items)})
        for item in self.items:
            self._append_event("item_update", item.to_dict())

    def update_item_checkpoint(self, item: BatchItem) -> None:
        self._append_event("item_update", item.to_dict())

    def update_status_checkpoint(self) -> None:
        self._append_event("batch_status", {"status": self.status})

    def run(self) -> Dict[str, Any]:
        total = len(self.items)
        interrupted = False
        self.init_checkpoint()

        for idx, item in enumerate(self.items, 1):
            if item.status == "completed":
                continue

            item.status = "in_progress"
            self.update_item_checkpoint(item)

            cmd = [sys.executable, str(self.cli_script), item.target, *self.extra_args]
            success = False

            for attempt in range(self.max_retries):
                t_start = time.time()
                try:
                    res = subprocess.run(cmd, cwd=self.root_dir, timeout=180)
                    item.duration_seconds += (time.time() - t_start)
                    if res.returncode == 0:
                        item.status = "completed"
                        item.error = None
                        success = True
                        break
                    else:
                        item.retries += 1
                        item.error = f"Exit code {res.returncode}"
                        if attempt < self.max_retries - 1:
                            time.sleep((2 ** attempt) + random.uniform(0.5, 1.5))
                except subprocess.TimeoutExpired:
                    item.duration_seconds += (time.time() - t_start)
                    item.retries += 1
                    item.error = "Timeout (180s)"
                    if attempt < self.max_retries - 1:
                        time.sleep((2 ** attempt) + random.uniform(1.0, 2.0))
                except KeyboardInterrupt:
                    interrupted = True
                    item.status = "pending"
                    item.error = "User Interrupted"
                    break

            if interrupted:
                self.status = "interrupted"
                self.update_status_checkpoint()
                break

            if not success:
                item.status = "failed"

            self.update_item_checkpoint(item)

            has_more = any(it.status != "completed" for it in self.items[idx:])
            if has_more and not interrupted:
                time.sleep(random.uniform(self.delay_range[0], self.delay_range[1]))

        if not interrupted:
            all_done = all(it.status == "completed" for it in self.items)
            self.status = "completed" if all_done else "finished_with_errors"
            self.update_status_checkpoint()

        return {
            "batch_id": self.batch_id,
            "status": self.status,
            "total": total,
            "completed": sum(1 for it in self.items if it.status == "completed"),
            "failed": sum(1 for it in self.items if it.status == "failed"),
            "pending": sum(1 for it in self.items if it.status in ("pending", "in_progress")),
        }
```

---

## 4. Configuration Matrix & Default Parameters

| Parameter | Standard Default | Recommended Range | Purpose |
| :--- | :--- | :--- | :--- |
| `MICRO_DELAY_MIN` | `2.0s` | `2.0s – 5.0s` | Minimum pause between target extractions |
| `MICRO_DELAY_MAX` | `5.0s` | `4.0s – 8.0s` | Maximum pause between target extractions |
| `MAX_SUBPROCESS_RETRIES` | `3 attempts` | `2 – 4 attempts` | Retry count for failed targets |
| `SUBPROCESS_TIMEOUT` | `180s` | `120s – 300s` | Process hang and dead connection safeguard |
| `MACRO_BATCH_INTERVAL` | `15 items` | `10 – 25 items` | Frequency of long stamina breaks |
| `MACRO_COOLDOWN_RANGE` | `30.0s – 60.0s`| `30.0s – 180.0s` | Extended human work session break |
| `BROWSER_RECYCLE_INTERVAL`| `25 items` | `20 – 40 items` | Browser context memory purging |

---

## 5. Verification & Audit Checklist for AI Coding Agents

Before approving or considering any PPA scraping tool or plugin complete, verify the implementation against this checklist:

- [ ] **Universal File Parsing**: Correctly parses `.txt`, `.csv` (auto column detect), and `.md` target links.
- [ ] **No Fixed Delays**: All pauses use bounded `random.uniform()` stochastic jitter.
- [ ] **O(1) JSONL Checkpointing**: Batch state writes to `batches/<batch_id>.jsonl` using Append-Only JSON event sourcing after every single target.
- [ ] **Exponential Backoff**: Failed subprocesses retry up to 3 times with `2 ** attempt` delay.
- [ ] **1-Command Resumption**: System supports `/resume-batch` and warns on startup if incomplete batches exist.
- [ ] **O(1) Dynamic Dispatch**: Tool execution uses `build_cmd_args()` and dictionary registry lookup.
- [ ] **EAFP Hardening**: Zero unhandled `except Exception: pass` statements across all Hub modules.
- [ ] **Decoupled Architecture**: Hub cleanly split into `ui.py`, `agent.py`, `handlers.py`, `batch_manager.py`, `storage.py`, and controller CLI.
- [ ] **Zero-Prerequisite Stealth Binaries**: Auto-verifies and auto-downloads missing browser binaries on first run without crashing.
