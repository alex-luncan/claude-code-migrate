<div align="center">

# 🔁 OSmigration

**Move a project between Windows and macOS so Claude Code just works on the other side.**

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that copies your project to a safe staging area, fixes everything OS‑specific it can, reports everything it can't, and hands you a single `PROJECTNAME_to-transfer.zip`.

[![Platform](https://img.shields.io/badge/platform-Windows%2011%20%E2%86%94%20macOS-blue)](#)
[![Python](https://img.shields.io/badge/python-3.8%2B%20(stdlib%20only)-3776AB?logo=python&logoColor=white)](#)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-skill-D97757)](#)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

</div>

---

## Why this exists

Moving a repo between operating systems usually works. Moving a repo **together with its Claude Code setup** usually doesn't:

- MCP servers wrapped in `cmd /c` fail on macOS; `npx` without the wrapper fails on Windows
- Hooks written for PowerShell never run in zsh, and `brew`/`open` don't exist in Git Bash
- `.sh` files with CRLF endings die with *bad interpreter*
- `CLAUDE.md` says "use PowerShell" and Claude obeys it on a Mac
- Absolute paths (`C:\Users\...`, `/Users/...`) hide in `.env`, `settings.local.json`, permission rules, MCP args
- Filenames with `:` or trailing dots can't even be extracted on Windows

OSmigration finds all of that, fixes the mechanical parts automatically, and leaves you a short, precise list for the rest.

## ✨ What it does

| | Automatic (applied to the copy) | Reported for manual review |
|---|---|---|
| **Claude Code config** | `cmd /c` wrapper added/removed on MCP servers, `python` ↔ `python3`, path separators in args | Hook commands, permission rules with absolute paths, `CLAUDE.md` wording |
| **Paths** | Every reference to the old project root rewritten to the new location (`--dest-root`) | Absolute paths that point *outside* the project |
| **Line endings** | Normalised to LF (CRLF kept for `.bat/.cmd/.ps1`), `.gitattributes` policy added | `core.autocrlf` in repo config |
| **Files** | Regenerable dirs skipped (`node_modules`, `.venv`, `dist`, `target`…), junk removed (`.DS_Store`, `Thumbs.db`) | Case‑only filename collisions |
| **macOS → Windows** | Invalid filenames renamed (`<>:"\|?*`, `CON`, trailing dots), symlinks replaced by copies | Long paths, `.sh` scripts |
| **Windows → macOS** | `.sh` / shebang files marked executable inside the zip | `.bat` / `.ps1` scripts |
| **Scripts** | — | `package.json` scripts, Makefile, CI steps using the wrong shell |
| **User‑level config** | Optional: global `CLAUDE.md`, slash commands, agents, MCP servers from `~/.claude.json` | — |

> 🔒 **The original project is never modified.** Every change happens in a temporary copy. Credentials are never copied.

## 📦 Output

Two files land in the directory you launched Claude from:

```
myapp_to-transfer.zip          ← the adapted project (+ MIGRATION_REPORT.md inside)
myapp_MIGRATION_REPORT.md      ← same report, for reading before you unzip
```

The report contains: what was changed, what still needs a decision (with file and line), and a checklist for the target machine.

<details>
<summary><b>Example report excerpt</b></summary>

```markdown
## Automatic changes applied to the staged copy (8)

- `.mcp.json` — $.mcpServers.fs: removed 'cmd /c' wrapper -> npx -y @modelcontextprotocol/server-filesystem
- `.env` — Replaced 1 occurrence(s) of the old project root with /Users/user-name/Projects/myapp
- `(project)` — Normalised line endings in 14 file(s): LF everywhere, CRLF for .bat/.cmd/.ps1.
- `.gitattributes` — Added end-of-line policy

## Manual actions required (3)

1. `.claude/settings.local.json` — Hook at $.hooks.PostToolUse[0].hooks[0] uses source-OS shell syntax:
   'powershell -c "npx prettier --write $env:FILE"' (PowerShell invocation)
   - _Rewrite for the target shell (see references/windows-to-macos.md, Hooks section)._
2. `package.json` — script "build": 'set NODE_ENV=production && tsc' (cmd-style 'set VAR=' env assignment)
   - _Use cross-platform tools (cross-env, rimraf, shx, npm-run-all) or rewrite for the target shell._
3. `CLAUDE.md` — Mentions source-OS specifics: powershell, c:\.
   - _Claude follows CLAUDE.md literally; rewrite these instructions for the target OS._
```

</details>

## 🚀 Installation

**Option A – as a Claude Code skill (recommended)**

```bash
git clone https://github.com/alex-luncan/claude-code-migrate.git
cp -r claude-code-migrate ~/.claude/skills/osmigration        # macOS
# or on Windows (PowerShell):
Copy-Item -Recurse claude-code-migrate "$env:USERPROFILE\.claude\skills\osmigration"
```

Restart `claude`. Install it on **both** machines, so you can migrate in either direction.

**Option B – project‑local**

Put the folder at `.claude/skills/osmigration/` inside a project.

**Option C – script only**

`scripts/migrate.py` is standalone Python 3.8+ with no dependencies. Use it without Claude at all (see below).

## 🧑‍💻 Usage with Claude Code

Just ask:

> "I'm moving this project to my Mac, prepare it for transfer."

Claude will ask for three things, then run the pipeline:

| Input | Example | Required |
|---|---|---|
| Source path | `.` or `C:\Users\user-name\Projects\myapp` | yes |
| Direction | `win2mac` or `mac2win` | yes – never guessed |
| Target project root | `/Users/user-name/Projects/myapp` | optional, enables automatic path rewriting |

It also asks whether to include your **user‑level** Claude config (global `CLAUDE.md`, custom commands, agents, user‑scope MCP servers). These live outside the repo and are the most common thing people forget.

The flow:

```
prepare  ──►  stage copy in temp  ──►  scan  ──►  auto‑fix  ──►  MIGRATION_REPORT.md
                                                                        │
              Claude edits the remaining manual items in the copy ◄─────┘
                                                                        │
package  ──►  PROJECTNAME_to-transfer.zip in your working directory ◄───┘
```

## 🖥️ Usage without Claude

```bash
# Windows → macOS, review step in between
python scripts/migrate.py prepare "C:\Users\user-name\Projects\myapp" \
    --direction win2mac \
    --dest-root /Users/user-name/Projects/myapp \
    --include-global
# ... edit the staged copy printed as STAGED_DIR ...
python scripts/migrate.py package --staged "<STAGED_DIR>"

# macOS → Windows, one shot
python3 scripts/migrate.py all ~/Projects/myapp \
    --direction mac2win \
    --dest-root C:/Users/user-name/Projects/myapp
```

<details>
<summary><b>All commands and flags</b></summary>

| Command | What it does |
|---|---|
| `stage` | copy the project into a temp staging dir |
| `scan` | detect OS‑specific content, write the report |
| `convert` | apply the automatic fixes to the staged copy |
| `package` | zip the staged copy → `<workdir>/<project>_to-transfer.zip`, delete temp |
| `prepare` | `stage` + `scan` + `convert`, then stop for manual edits |
| `all` | everything in one go |

| Flag | Purpose |
|---|---|
| `--direction win2mac\|mac2win` | required for `stage` / `prepare` / `all` |
| `--dest-root PATH` | where the project will live on the target; enables path rewriting |
| `--source-root PATH` | how the project root is spelled inside its files if different from the real path (mapped drive, junction, WSL `/mnt/c/...`) |
| `--include-global` | collect `~/.claude/{CLAUDE.md,settings.json,commands,agents,skills}` and MCP servers from `~/.claude.json` into `_claude-global-extras/` |
| `--exclude DIR` / `--keep DIR` | adjust the list of skipped directories |
| `--exclude-git` | leave `.git` out of the archive |
| `--name NAME` | archive name other than the folder name |
| `--workdir DIR` | where to write the archive (default: current directory) |
| `--keep-staging` | don't delete the temp copy after packaging |

</details>

## 📋 On the target machine

The report ends with a full checklist. The short version:

1. Unzip
2. Reinstall dependencies (`npm ci`, `uv sync`, `cargo build`… the report tells you which)
3. `claude` → log in again (credentials never travel)
4. Restore `_claude-global-extras/` if you included it (its README explains where each file goes; MCP servers via `claude mcp add-json`)
5. Work through *Manual actions* in the report
6. `claude` → `/doctor`

## 🗂️ Repository layout

```
claude-code-migrate/
├── SKILL.md                          # instructions Claude follows
├── scripts/
│   └── migrate.py                    # the whole pipeline, stdlib only
└── references/
    ├── windows-to-macos.md           # translation tables: hooks, scripts, env vars, config locations
    └── macos-to-windows.md
```

## 🤔 Design decisions

- **LF in both directions.** Windows tooling handles LF fine; CRLF shell scripts break on macOS. Only `.bat/.cmd/.ps1` stay CRLF, enforced by `.gitattributes`.
- **Forward slashes for Windows paths.** `C:/Users/user-name/...` works in JSON configs, Node, Python, Git and Claude Code, and needs no escaping.
- **Hooks are never auto‑translated.** A wrong hook fails silently or blocks every edit. They're flagged with the exact command and a translation table instead.
- **`node_modules` is never copied**, even on request. Native binaries are platform‑specific; the lockfile regenerates them exactly.
- **Git Bash awareness.** Claude Code on Windows runs bash commands through Git Bash, so bash‑style hooks are reported as *info*, not errors, when going mac → win.

## 🤝 Contributing

Issues and PRs welcome. Useful contributions:

- more translation rows in `references/`
- detection rules for additional toolchains (Gradle, .NET, Deno…)
- a `linux` direction

Test with a throwaway project containing the things you want detected, run `prepare`, and read the report.

## 📄 License

MIT — see [LICENSE](LICENSE).
