# macOS → Windows translation reference

Read this when fixing the *Manual actions* items for a `mac2win` migration. Edit files in the
staged copy only.

## Where Claude Code keeps things

| Item | macOS | Windows |
|---|---|---|
| User settings, commands, agents, global CLAUDE.md | `~/.claude/` | `C:\Users\<you>\.claude\` |
| Global config, user-scope + local-scope MCP servers | `~/.claude.json` | `C:\Users\<you>\.claude.json` |
| Project settings | `.claude/settings.json`, `.claude/settings.local.json` | same |
| Project MCP servers | `.mcp.json` | `.mcp.json` |
| Credentials | macOS Keychain | Windows Credential Manager (log in again) |
| Shell used for Bash tool and hooks | zsh/bash | **Git Bash** (must be installed; PowerShell is not used by the Bash tool) |

Because Claude Code on Windows drives Git Bash, bash-style hooks and `.sh` scripts often keep
working as long as they avoid mac-only commands. Do not convert them to PowerShell unless the
user wants native Windows scripts.

## MCP server entries

The script already wraps npm shims in `cmd /c` and swaps `python3` → `python`. Check what's left:

| macOS | Windows |
|---|---|
| `"command": "npx", "args": ["-y", "pkg"]` | `"command": "cmd", "args": ["/c", "npx", "-y", "pkg"]` (npx is a `.cmd` shim; without `cmd /c` it fails to spawn) |
| `"command": "node"` | `"command": "node"` (real .exe, fine) |
| `"command": "python3"` | `"command": "python"` (the Windows launcher rarely provides `python3`) |
| `"command": "uvx"` / `"uv"` | same (real .exe) |
| `"command": "/opt/homebrew/bin/xyz"` | `"command": "xyz"` on PATH, or full path with forward slashes `C:/tools/xyz.exe` |
| `"env": {"PATH": "/opt/homebrew/bin:$PATH"}` | remove, or set with `;` separators and no `$PATH` expansion |
| `~/something` in args | full path; `~` is not expanded in MCP configs |

Forward slashes (`C:/Users/user-name/...`) are valid in JSON configs on Windows and avoid escaping.

## Hooks (`.claude/settings*.json` → `hooks`)

Hooks run through Git Bash on Windows, so most bash stays. Replace what Git Bash lacks:

| macOS-only | Works on Windows (Git Bash) |
|---|---|
| `open file` / `open url` | `start "" file` (cmd) or `explorer.exe`, or `cmd //c start "" url` from Git Bash |
| `brew install x` | `winget install x` / `choco install x` / `scoop install x` |
| `sed -i '' 's/a/b/' f` | `sed -i 's/a/b/' f` (GNU sed syntax) |
| `pbcopy` / `pbpaste` | `clip` / `powershell Get-Clipboard` |
| `say "done"` | `powershell -c "[console]::beep()"` or drop |
| `osascript ...` | drop or `powershell -c ...` |
| `/usr/bin/env python3` | `python` |
| `jq` | install with `winget install jqlang.jq` |
| `chmod +x` | no-op on Windows; remove |
| `sudo x` | remove; run terminal as admin if truly needed |

If the user prefers native PowerShell hooks, prefix with `powershell -NoProfile -c "..."` and use
`$env:VAR` for environment variables.

## `package.json` scripts and other command strings

| macOS/bash | Cross-platform replacement |
|---|---|
| `NODE_ENV=production tsc` | `cross-env NODE_ENV=production tsc` |
| `rm -rf dist` | `rimraf dist` |
| `cp -r a b` / `mkdir -p d` | `shx cp -r a b` / `shx mkdir -p d` |
| `export VAR=x && cmd` | `cross-env VAR=x cmd` |
| `cmd1 & cmd2` | `npm-run-all -p cmd1 cmd2` or `concurrently` |
| `$npm_package_version` | `%npm_package_version%` in cmd; or keep `$` and use `cross-var` |
| `open http://localhost:3000` | `open-cli http://localhost:3000` |
| `./scripts/x.sh` | `bash scripts/x.sh` (works in Git Bash and in npm on Windows if Git Bash is npm's script-shell) |

npm on Windows runs scripts through `cmd.exe` by default. Either use the cross-platform tools
above or set `npm config set script-shell "C:\Program Files\Git\bin\bash.exe"` on the target.

## Scripts and file permissions

- `.sh` files run under Git Bash (`bash scripts/x.sh`) or WSL. The executable bit does not exist
  on NTFS; nothing to do.
- `#!/usr/bin/env python3` shebangs are ignored on Windows; invoke with `python script.py`.
- Consider adding a `.ps1` twin for scripts developers will run from PowerShell.

## Filenames

The script renamed anything Windows rejects (`<>:"|?*`, trailing dots/spaces, `CON`, `NUL`,
`COM1`…). Update references to those names. Also check for paths deeper than ~240 characters;
on the target run `git config core.longpaths true` or enable
`HKLM\SYSTEM\CurrentControlSet\Control\FileSystem\LongPathsEnabled`.

Symlinks were replaced with copies. If a link is essential, recreate it with
`mklink` (needs Developer Mode or admin) after extracting.

## Environment variables

- `~/.zshrc` `export VAR=value` → *Settings → System → Advanced → Environment Variables*, or
  `[Environment]::SetEnvironmentVariable("VAR","value","User")` in PowerShell, or `setx VAR value`.
- `$HOME` → `%USERPROFILE%` / `$env:USERPROFILE`; `~/Library/Application Support` → `%APPDATA%`;
  `~/Library/Caches` → `%LOCALAPPDATA%`; `$TMPDIR` → `%TEMP%`.
- PATH lists use `;` not `:`.

## CLAUDE.md phrasing

Replace "brew install", "zsh", "`~/...`" paths and "chmod +x" with Windows equivalents. State
that the Bash tool runs in Git Bash so Claude doesn't reach for PowerShell-only cmdlets in
bash commands (or vice versa).

## Permission rules

`Read(/Users/user-name/**)` → `Read(~/**)` or `Read(./**)`; keep forward slashes.

## Git

- `.gitattributes` with `* text=auto eol=lf` was added. On the target run
  `git config core.autocrlf false` so Git does not turn the working tree into CRLF.
- Case-only filename collisions must be resolved before extracting on NTFS.

## Node / Python toolchain

- Install Git for Windows (Git Bash) first; Claude Code depends on it.
- Node: `fnm` or `nvm-windows`; native modules need the Visual Studio Build Tools
  (`npm install -g windows-build-tools` is deprecated; use the VS installer, "Desktop development
  with C++").
- Python: use the python.org installer or `winget install Python.Python.3.12`; recreate the venv
  (`python -m venv .venv; .venv\Scripts\activate`).
- Docker Desktop on Windows uses WSL2; `docker-compose` volume paths like `./data:/data` are fine,
  absolute `/Users/...` mounts are not.
