# PPA Framework — Agent Guide

> **Purpose**: Single entry point for any agent or human **updating, creating, moving, or deleting** documents in `.framework/PPA/`. Read this file first. Its job is to keep every document connected to the framework tree so no file ever becomes a stranger to the system.
> **Root Document**: [plugin_project_architecture.md](plugin_project_architecture.md)

---

## 1. Framework Tree (Current State)

```text
agent.md                        ← YOU ARE HERE (maintenance protocol, not part of the spec tree)
│
plugin_project_architecture.md  ← ROOT (Hub-and-Spoke concept — everything derives from this)
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

## 2. Document Registry

| File | Node | Role | Upstream | Downstream / Related |
| :--- | :--- | :--- | :--- | :--- |
| `plugin_project_architecture.md` | ROOT | Defines the Hub-and-Spoke concept; directory of the whole framework | — (root) | All 5 documents below |
| `cli_architecture.md` | Hub | Universal CLI/TUI standard: visual system, autocomplete, packaging, session commands, Root README blueprint | Root | `agentic_cli_architecture.md` (extends it), `deployment_architecture.md`, `bulk_scraper_engine_specification.md` |
| `agentic_cli_architecture.md` | Hub | Agentic extension: AI mode, multi-turn sessions, OpenRouter tool calling, command matrix | `cli_architecture.md`, Root | `bulk_scraper_engine_specification.md` (tool dispatch, `/batch`, `/resume-batch`), `plugin_sdk_specification.md` (tool schemas ↔ `SKILL.md`) |
| `bulk_scraper_engine_specification.md` | Hub | Batch & resilience engine: 15 patterns, `batch_manager.py` reference, audit checklist | Root | `agentic_cli_architecture.md`, `cli_architecture.md` (Pattern 15 layout), `plugin_sdk_specification.md` (Pattern 5 decoupling) |
| `deployment_architecture.md` | Cross-cutting | Packaging, dual-mode distribution, `storage.py` path resolution, bootstrap | Root | `cli_architecture.md` (packaging), `plugin_sdk_specification.md` (layout/`Outputs/`), `bulk_scraper_engine_specification.md` (`batches/`) |
| `plugin_sdk_specification.md` | Spoke | The standard every plugin implements: `core.py`, `cli.py`, `SKILL.md`, `Outputs/` | Root | `agentic_cli_architecture.md`, `bulk_scraper_engine_specification.md`, `deployment_architecture.md` |

---

## 3. Connection Rules (How Files Stay Connected)

1. **Navigation header is mandatory.** Every framework document must start with the standard blockquote nav block directly after its title/intro:
   - `> **PPA Framework**: <layer> — <one-line role>`
   - `> **Upstream (Root)**: [plugin_project_architecture.md](plugin_project_architecture.md)` (the root document instead lists its children)
   - `> **Related**: <sibling/cross-cutting links with a short "why">`
   - `> **Framework Map & Maintenance Protocol**: [agent.md](agent.md)` is referenced from the root document; the root is referenced from all others.
2. **Links are always relative.** Same-folder Markdown links only — never absolute `file:///` paths, never old project locations. Stale absolute paths are a defect.
3. **Every link is bidirectional.** If document A references document B, then B's nav block (or body) must reference A back. One-directional links break the tree.
4. **The tree map lives in two places and must match**: the `Framework Document Map` section in the root document, and Section 1 of this file. Update both or neither.
5. **Section references cite by name**, e.g. "Pattern 13", "the 7-Section README blueprint" — not by fragile line numbers or anchor URLs.
6. **No content duplication across documents.** A document links to the canonical source instead of restating it (same principle as the Hub linking to plugin READMEs).

---

## 4. Updating an Existing File — Checklist

- [ ] Read the file (and its Registry row above) before editing — never assume its current state.
- [ ] Keep the nav block intact and accurate; if the edit changes the document's role or relationships, update the nav block accordingly.
- [ ] If you add a concept that another document already owns, **link to it** instead of rewriting it.
- [ ] If you remove or rename a concept that other documents reference, grep the folder for that concept and update every reference.
- [ ] Verify every relative link you touched still resolves (see Section 7).
- [ ] If the change affects the tree (new role, new relationship), update Section 1 here **and** the Framework Document Map in the root document.

## 5. Creating a New File — Protocol

1. **Classify its node first.** Decide where it belongs:
   - *Hub concern* (TUI, agent, batch, storage, packaging) → child of the root, sibling of the other Hub docs.
   - *Spoke concern* (plugin internals) → child of the root in the Spokes branch.
   - *Cross-cutting* (applies to several nodes) → note it as cross-cutting in the nav block.
   - If it doesn't fit the PPA framework, it does not belong in `.framework/PPA/`.
2. **Create the file with the mandatory nav block** (Section 3, rule 1), including a one-line role and at minimum the upstream root link.
3. **Wire the backlinks**: add the new file to the related/child lists of every document it relates to (at minimum: the root document).
4. **Register it**: add a row to the Document Registry (Section 2) and a node to the tree in Section 1 here, plus the Framework Document Map in the root document.
5. **Verify links** (Section 7).

Result: the new file is born connected — reachable from the root, linked to its siblings, and registered in the map. It never enters the system as a stranger.

## 6. Renaming, Moving, or Deleting a File

1. Grep `.framework/PPA/` for the filename to find every inbound link.
2. Update all inbound links **before or together with** the rename/move/delete.
3. Update the tree (Section 1), the Registry (Section 2), and the root document's Framework Document Map.
4. If deleting: remove its row/node everywhere and fold any unique responsibilities it owned into the nearest surviving document, or explicitly note the removal.

## 7. Link Integrity Check

After any structural change, verify every relative `.md` link inside `.framework/PPA/` resolves to an existing file in the same folder. Minimal check (PowerShell):

```powershell
Get-ChildItem -Path . -Filter "*.md" | Select-String -Pattern '\]\(([^)#]+\.md)\)' | ForEach-Object {
  $dir = Split-Path $_.Path; $target = $_.Matches[0].Groups[1].Value
  if ($target -notmatch '^https?://' -and -not (Test-Path (Join-Path $dir $target))) { "BROKEN: $($_.Path) -> $target" }
}
```

No output means every link resolves.
