# Windows → macOS translation reference

Read this when fixing the *Manual actions* items for a `win2mac` migration. Edit files in the
staged copy only.

## Where Claude Code keeps things

| Item | Windows | macOS |
|---|---|---|
| User settings, commands, agents, global CLAUDE.md | `C:\Users\<you>\.claude\` | `~/.claude/` |
| Global config, user-scope + local-scope MCP servers | `C:\Users\<you>\.claude.json` | `~/.claude.json` |
| Project settings | `.claude\settings.json`, `.claude\settings.local.json` | same, forward slashes |
| Project MCP servers | `.mcp.json` | `.mcp.json` |
| Credentials | Windows Credential Manager | macOS Keychain (log in again) |
| Shell used for Bash tool and hooks | Git Bash | `/bin/zsh` or `/bin/bash` |

## MCP server entries

The script already strips `cmd /c` and `.cmd`/`.exe` suffixes. Check what's left:

| Windows | macOS |
|---|---|
| `"command": "cmd", "args": ["/c", "npx", ...]` | `"command": "npx", "args": [...]` |
| `"command": "python"` | `"command": "python3"` (or the venv's `python`) |
| `"command": "C:\\Users\\user-name\\AppData\\Roaming\\npm\\node.exe"` | `"command": "node"` (rely on PATH) |
| `"command": "uvx"` | `"command": "uvx"` (install uv with brew) |
| `"env": {"HOME": "C:\\Users\\user-name"}` | usually drop; zsh sets HOME |
| args with `C:\\...` outside the project | equivalent `~/...` path or `${HOME}`-free absolute path |

Note: MCP `command` is not run through a shell, so `~` and `$HOME` are **not** expanded in
`command`/`args`. Use a full path or rely on PATH.

## Hooks (`.claude/settings*.json` → `hooks`)

Hooks run through the user's shell. PowerShell syntax has to be rewritten.

| PowerShell / cmd | bash / zsh |
|---|---|
| `powershell -c "..."` / `pwsh -c "..."` | drop the wrapper; write the command directly |
| `$env:CLAUDE_FILE_PATH` | `$CLAUDE_FILE_PATH` |
| `%CLAUDE_PROJECT_DIR%` | `$CLAUDE_PROJECT_DIR` |
| `Get-Content file` / `type file` | `cat file` |
| `Write-Output "x"` / `echo x` | `echo x` |
| `Test-Path file` | `[ -e file ]` |
| `Remove-Item -Recurse -Force dir` / `rmdir /s /q dir` | `rm -rf dir` |
| `Copy-Item a b` / `copy a b` | `cp a b` |
| `Select-String pattern` / `findstr pattern` | `grep pattern` |
| `; ` between commands (PowerShell) | `; ` or `&&` |
| `Start-Process x` / `start x` | `open x` |
| `exit 2` (blocking hook) | `exit 2` (same) |

Hook stdin (JSON event payload) is identical on both platforms; `jq` is available after
`brew install jq` and is the usual way to read it.

## `package.json` scripts and other command strings

| Windows-only | Cross-platform replacement |
|---|---|
| `set NODE_ENV=production && tsc` | `cross-env NODE_ENV=production tsc` |
| `rmdir /s /q dist` / `del /q x` | `rimraf dist` |
| `copy a b` / `xcopy /s a b` | `shx cp -r a b` or `cpy-cli` |
| `mkdir dir` (no error if exists on Windows) | `mkdirp dir` or `shx mkdir -p dir` |
| `cmd1 & cmd2` | `npm-run-all -p cmd1 cmd2` or `concurrently` |
| `%npm_package_version%` | `$npm_package_version` |
| `.\scripts\x.js` | `./scripts/x.js` or `node scripts/x.js` |
| `start http://localhost:3000` | `open http://localhost:3000` (mac only) or `open-cli` |

Prefer the cross-platform tools when the project will keep living on both OSes; add them as
devDependencies.

## Scripts

- `.bat` / `.cmd` / `.ps1` do not run on macOS. Write an equivalent `.sh` (start with
  `#!/usr/bin/env bash` and `set -euo pipefail`) and mark it executable (`chmod +x`).
- Any existing `.sh` with CRLF was fixed by the script.
- `python` → `python3`; `py -3` → `python3`; `pip` → `pip3` or `python3 -m pip`.

## Environment variables

- PowerShell profile (`$PROFILE`) or System Properties → `~/.zshrc` with `export VAR=value`.
- `%USERPROFILE%` → `$HOME`, `%APPDATA%` → `~/Library/Application Support`,
  `%LOCALAPPDATA%` → `~/Library/Caches` or `~/Library/Application Support`, `%TEMP%` → `$TMPDIR`.
- Path lists use `:` not `;` (`PATH=$PATH:/new/dir`).

## CLAUDE.md phrasing

Replace instructions like "use PowerShell", "run `.\build.ps1`", "paths under `C:\...`" with the
macOS equivalents. Mention Homebrew if tools are installed that way. Keep it factual; Claude will
follow it literally.

## Permission rules (`permissions.allow` / `deny`)

`Read(C:\Users\user-name\**)` → `Read(~/**)` or `Read(./**)`. Always forward slashes. Bash rules
(`Bash(npm run *)`) are unchanged.

## Git

- `.gitattributes` with `* text=auto eol=lf` was added; on the Mac run `git config core.autocrlf input`.
- If `core.longpaths` was set, it can be dropped.
- Case-only renames: `git mv Readme.md tmp && git mv tmp readme.md`.

## Node / Python toolchain

- Install `fnm` or `nvm`; `nvm-windows` config does not transfer. Add `.nvmrc` if missing.
- Native modules (`node-gyp`, `sharp`, `better-sqlite3`) rebuild on `npm ci`; Xcode command line
  tools (`xcode-select --install`) are needed.
- Python: recreate the venv (`python3 -m venv .venv && source .venv/bin/activate`).
