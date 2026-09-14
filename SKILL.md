---
name: osmigration
description: Move a software project between Windows and macOS (either direction) so that Claude Code and the project "just work" on the other machine. Use this whenever the user mentions transferring, migrating, moving, copying or porting a project, repo, codebase or workspace from Windows to a Mac or from a Mac to Windows, switching laptops/OS, or asks what will break in Claude Code (MCP servers, hooks, settings, CLAUDE.md, line endings, paths) after a platform change. It never modifies the original project; it works on a copy in a temp directory and produces a PROJECTNAME_to-transfer.zip archive plus a migration report in the current working directory.
---

# OSmigration

Prepare a project for a move between Windows 11 and macOS without touching the original.
The script stages a full copy in a temp directory, scans it for OS-specific content, applies
the safe automatic fixes there, and packages the result as `<project>_to-transfer.zip` in the
work directory (where Claude was launched, unless the user says otherwise). A
`MIGRATION_REPORT.md` is written inside the archive and next to it.

The original project directory is read-only for this skill. Never edit, rename, or delete anything
under the source path. All edits happen in the staged copy.

## Workflow

### 1. Gather the three inputs (ask if not given)

- **Source path** of the project on the current machine.
- **Direction**: `win2mac` (Windows → macOS) or `mac2win` (macOS → Windows). The user must state
  it; do not guess from the current OS, since people sometimes run this on the destination machine
  against a mounted drive.
- **Target project root** (`--dest-root`), e.g. `/Users/user-name/Projects/myapp` or
  `C:/Users/user-name/Projects/myapp`. Optional, but with it the script rewrites every absolute reference
  to the project root automatically. Ask once; if the user doesn't know, proceed without it and the
  paths appear as manual items in the report.

Also ask whether to include user-level Claude Code config (`--include-global`): global CLAUDE.md,
custom slash commands, agents, and user-scoped MCP servers from `~/.claude.json`. These live
outside the repo and are the most common thing people forget. Credentials are never copied.

### 2. Run `prepare` (stage + scan + convert), not `all`

```bash
python scripts/migrate.py prepare "<source>" --direction win2mac \
    --dest-root /Users/user-name/Projects/myapp --include-global
```

`prepare` stops after converting so the manual items can be handled before packaging. On Windows
use `python`; on macOS `python3` (stdlib only, Python 3.8+). The command prints `STAGED_DIR=...`
and `REPORT=...`; read the report.

What the script does automatically in the staged copy:

- skips `node_modules`, `.venv`, `dist`, `build`, `target`, `__pycache__` and similar (they're
  regenerated on the target) and drops `.DS_Store`/`Thumbs.db` junk
- normalises line endings to LF (CRLF kept only for `.bat/.cmd/.ps1`) and adds a `.gitattributes`
  policy so Git stops flipping them
- MCP servers in `.mcp.json` / `.claude/*.json`: removes or adds the `cmd /c` wrapper that npm
  shims need on Windows, swaps `python`↔`python3`, fixes separators in args
- rewrites the old project root to `--dest-root` everywhere (configs, `.env`, CLAUDE.md, code)
- mac→win: renames files with characters/names Windows rejects, replaces symlinks with copies
- win→mac: marks `.sh` and shebang files executable inside the zip

What it only reports (needs judgement): hook commands, `package.json` scripts, Makefiles/CI
steps, CLAUDE.md instructions, permission rules, absolute paths outside the project, filename
case collisions, `.bat`/`.ps1` scripts.

### 3. Work the manual list in the staged copy

Open the report's *Manual actions required* section and fix each item by editing files under
`STAGED_DIR` (never the source). Use the reference for the direction:

- `references/windows-to-macos.md` for `win2mac`
- `references/macos-to-windows.md` for `mac2win`

They contain the translation tables for hooks, npm scripts, shell commands, env vars, and the
Claude Code config locations on each OS. Typical edits:

- rewrite hook commands for the target shell (PowerShell ↔ bash/zsh)
- replace `set VAR=x && cmd` / `VAR=x cmd` with `cross-env`, `rm -rf` with `rimraf`, etc.
- rewrite CLAUDE.md lines like "use PowerShell" or "install with brew"
- turn `Read(C:\Users\...)`-style permission rules into `~/`-relative or project-relative globs
- resolve case-only filename collisions (pick one; both OSes are case-insensitive by default)

If an item is something only the user can decide (an absolute path to a tool that may not exist
on the other machine), leave it and say so; the report travels with the archive so they can finish
it on the target. Show the user a short summary of what you changed and what remains.

### 4. Package

```bash
python scripts/migrate.py package --staged "<STAGED_DIR>"
```

Writes `<project>_to-transfer.zip` and `<project>_MIGRATION_REPORT.md` to the current directory
(`--workdir` to change), then deletes the temp staging dir (`--keep-staging` to retain it). Tell
the user the archive path and the top three things to do on the target machine (the report's
final checklist has the full list: reinstall deps, log in to `claude` again, restore extras,
`/doctor`).

### Direct one-shot

If the user explicitly wants no review step, `all` does stage + scan + convert + package. Still
read the report afterwards and relay the manual items.

## Options worth knowing

| Flag | Use when |
|---|---|
| `--exclude DIR` / `--keep DIR` | project has other big regenerable dirs, or needs one of the default-excluded dirs |
| `--exclude-git` | user only wants the working tree, not history |
| `--source-root` | files reference the project by a different spelling than its real path (mapped drive, junction, WSL `/mnt/c/...`) |
| `--name` | archive should be named differently from the folder |

## Things to keep straight

- Line endings go to LF in both directions. Windows tooling is fine with LF; CRLF `.sh` files break
  on macOS with "bad interpreter". Only `.bat/.cmd/.ps1` stay CRLF.
- On Windows Claude Code runs commands through Git Bash, so bash-style hooks and `.sh` scripts can
  still work there if they avoid mac-only commands (`brew`, `open`, `sed -i ''`). PowerShell hooks
  never work on macOS.
- `~/.claude.json` (global) holds user-scope MCP servers and per-project local-scope servers; only
  `.mcp.json` in the repo travels with the project. `--include-global` extracts the rest.
- Do not copy `node_modules` or a virtualenv even if the user asks to "copy everything"; explain
  that native binaries in them are platform-specific and the lockfile regenerates them exactly.
- Auth never transfers (Keychain on macOS, Credential Manager on Windows); the user logs in again.
