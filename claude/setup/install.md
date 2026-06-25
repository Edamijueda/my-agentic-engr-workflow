# Install + update Claude Code

Cross-OS install playbook + update behavior + version pinning + binary integrity. Distilled from `https://code.claude.com/docs/en/setup`.

## Mental model

Six install paths, **three update behaviors**:

| Path | Auto-updates? | Notes |
| --- | --- | --- |
| Native installer (recommended) | **Yes**, in background | macOS / Linux / WSL / Windows. Direct from `claude.ai`. |
| Homebrew | No (manual `brew upgrade`) | macOS. Choose channel by cask name. |
| WinGet | No (manual `winget upgrade`) | Windows. |
| Linux package managers (apt / dnf / apk) | No (system upgrade workflow) | Signed repos. |
| npm | No (manual `npm install -g …@latest`) | Cross-platform. Installs the same native binary. |
| Desktop app | Self-managed | macOS / Windows GUI option. |

> Set `CLAUDE_CODE_PACKAGE_MANAGER_AUTO_UPDATE=1` to have Claude Code run the `brew upgrade` / `winget upgrade` for you in the background.

## System requirements

| | |
| --- | --- |
| **macOS** | 13.0+ |
| **Windows** | 10 1809+ or Server 2019+ |
| **Linux** | Ubuntu 20.04+, Debian 10+, Alpine 3.19+ |
| **Hardware** | 4 GB RAM, x64 or ARM64 |
| **Shell** | Bash, Zsh, PowerShell, or CMD |
| **Network** | Internet required (see `https://code.claude.com/docs/en/network-config`) |
| **Location** | Anthropic-supported countries |

`ripgrep` is usually bundled. If search fails, see the troubleshooting page.

## Quick install — recommended path per OS

### macOS / Linux / WSL

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

### Windows PowerShell

```powershell
irm https://claude.ai/install.ps1 | iex
```

### Windows CMD

```batch
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

**Which shell am I in on Windows?**

| Prompt looks like | Shell |
| --- | --- |
| `PS C:\Users\...` | PowerShell — use the `irm` command |
| `C:\Users\...` (no `PS`) | CMD — use the `curl` command |

> `'irm' is not recognized` → you're in CMD, not PowerShell.
> `The token '&&' is not a valid statement separator` → you're in PowerShell, not CMD.

Run `claude` after install. No admin privileges needed.

## Picking an install method

| Want… | Pick |
| --- | --- |
| Hands-off auto-updates | **Native installer** |
| Mac with Homebrew already set up | `brew install --cask claude-code` (stable) or `claude-code@latest` |
| Windows with WinGet | `winget install Anthropic.ClaudeCode` |
| Linux with apt/dnf/apk discoverability | Package manager repos (signed) |
| Cross-platform automation / CI / dev container | npm |
| GUI instead of terminal | Desktop app |

## Windows specifics

Three options. Pick based on where your projects live and whether you need sandboxing:

| Option | Requires | Sandboxing | When to use |
| --- | --- | --- | --- |
| **Native Windows** | Nothing (Git for Windows optional) | **No** | Windows-native projects and tools |
| **WSL 2** | WSL 2 enabled | **Yes** | Linux toolchains or sandboxed Bash |
| **WSL 1** | WSL 1 enabled | No | WSL 2 unavailable |

### Native Windows shell behavior

- **Without Git for Windows**: Claude Code uses the PowerShell tool for shell commands.
- **With Git for Windows**: Claude Code uses Git Bash for the Bash tool.
- If Claude Code can't find Git Bash, set the path explicitly in `settings.json`:
  ```json
  {
    "env": {
      "CLAUDE_CODE_GIT_BASH_PATH": "C:\\Program Files\\Git\\bin\\bash.exe"
    }
  }
  ```
- With Git for Windows installed, the PowerShell tool is rolling out progressively as an *additional* option. Opt in: `CLAUDE_CODE_USE_POWERSHELL_TOOL=1`. Opt out: `=0`. (See `../tools/tools-reference.md`.)

### WSL

Open your WSL distro and run the Linux installer (`curl https://claude.ai/install.sh | bash`). Install and launch `claude` inside the WSL terminal — not from PowerShell or CMD.

## Alpine / musl-based distributions

Extra deps needed:

```sh
apk add libgcc libstdc++ ripgrep
```

Then set in `settings.json`:

```json
{
  "env": {
    "USE_BUILTIN_RIPGREP": "0"
  }
}
```

## Verify install

```bash
claude --version
claude doctor   # detailed health check
```

If `claude --version` reports `command not found`, see the troubleshoot page.

## How auto-updates work

Native installations check on startup and periodically while running. Updates download in the background and apply **on next start**. `claude doctor` shows the result of the most recent attempt.

**npm gotcha**: if the npm global directory isn't writable, auto-update fails silently and Claude Code shows a one-time notice. Fix the permissions or move to the native installer.

**Homebrew / WinGet auto-run**: setting `CLAUDE_CODE_PACKAGE_MANAGER_AUTO_UPDATE=1` lets Claude Code run the upgrade in the background and show a restart prompt. Only updates the Claude Code package, not other software.

- **WinGet caveat**: upgrade may fail while Claude Code is running (Windows locks the exe). Falls back to showing the manual command.
- **apt / dnf / apk** still need manual upgrade (elevated privileges).
- **Homebrew tip**: `brew cleanup` periodically — old versions stay on disk.

**Known issue**: Claude Code may notify of updates before the new version is available in package managers. If upgrade fails, wait and retry.

### Channels

| Channel | Speed | Set via |
| --- | --- | --- |
| `latest` (default) | New releases as they ship | `autoUpdatesChannel: "latest"` |
| `stable` | ~1 week behind, skips major-regression releases | `autoUpdatesChannel: "stable"` |

Configure via `/config` → Auto-update channel, or in `settings.json`:

```json
{ "autoUpdatesChannel": "stable" }
```

**Homebrew is different**: cask name picks the channel (`claude-code` = stable, `claude-code@latest` = latest). The `autoUpdatesChannel` setting is ignored for Homebrew installs.

### Version pinning

| Setting | Effect | Scope |
| --- | --- | --- |
| `minimumVersion` | Floor — auto-update + `claude update` refuse to install below this | Any scope (user / project / managed) |
| `requiredMinimumVersion` | Refuse to start below this version | **Managed only** |
| `requiredMaximumVersion` | Refuse to start above this. Updates also respect this ceiling | **Managed only** |

When you switch from `latest` to `stable` via `/config` while on a newer build, you get prompted to either stay (sets `minimumVersion` to current) or allow downgrade. Switching back to `latest` clears it.

### Disable auto-updates

| Env var | Effect |
| --- | --- |
| `DISABLE_AUTOUPDATER=1` | Stops background check. **`claude update` and `claude install` still work.** |
| `DISABLE_UPDATES=1` | Blocks all update paths, including manual. Use when distributing through your own channels. |

Set in `env` block of `settings.json`:

```json
{ "env": { "DISABLE_AUTOUPDATER": "1" } }
```

## Manual update

```bash
claude update
```

Works even when `DISABLE_AUTOUPDATER=1` is set.

## Install a specific version or channel

Native installer accepts an arg:

```bash
# Latest (default)
curl -fsSL https://claude.ai/install.sh | bash

# Stable
curl -fsSL https://claude.ai/install.sh | bash -s stable

# Specific version
curl -fsSL https://claude.ai/install.sh | bash -s 2.1.89
```

**The channel chosen at install time becomes your default for auto-updates** — `bash -s stable` puts you on stable for all future auto-updates.

Windows PowerShell:
```powershell
& ([scriptblock]::Create((irm https://claude.ai/install.ps1))) stable
& ([scriptblock]::Create((irm https://claude.ai/install.ps1))) 2.1.89
```

Windows CMD:
```batch
... && install.cmd stable && ...
... && install.cmd 2.1.89 && ...
```

## Linux package manager setup

All repos are signed. **Verify the GPG fingerprint before trusting:**

```
31DD DE24 DDFA B679 F42D 7BD2 BAA9 29FF 1A7E CACE
```

apk key SHA256:
```
395759c1f7449ef4cdef305a42e820f3c766d6090d142634ebdb049f113168b6
```

### apt (Debian / Ubuntu) — stable channel

```bash
sudo install -d -m 0755 /etc/apt/keyrings
sudo curl -fsSL https://downloads.claude.ai/keys/claude-code.asc \
  -o /etc/apt/keyrings/claude-code.asc
echo "deb [signed-by=/etc/apt/keyrings/claude-code.asc] https://downloads.claude.ai/claude-code/apt/stable stable main" \
  | sudo tee /etc/apt/sources.list.d/claude-code.list
sudo apt update
sudo apt install claude-code

# Verify key
gpg --show-keys /etc/apt/keyrings/claude-code.asc
```

For `latest`: replace both the URL path **and** the suite name (the trailing word):
```
deb ... https://downloads.claude.ai/claude-code/apt/latest latest main
```

Upgrade: `sudo apt update && sudo apt upgrade claude-code`

### dnf (Fedora / RHEL) — stable channel

```bash
sudo tee /etc/yum.repos.d/claude-code.repo <<'EOF'
[claude-code]
name=Claude Code
baseurl=https://downloads.claude.ai/claude-code/rpm/stable
enabled=1
gpgcheck=1
gpgkey=https://downloads.claude.ai/keys/claude-code.asc
EOF
sudo dnf install claude-code
```

For `latest`: change `baseurl` to `https://downloads.claude.ai/claude-code/rpm/latest`.

dnf prompts to confirm fingerprint on first install — verify against `31DD … CACE`.

Upgrade: `sudo dnf upgrade claude-code`

### apk (Alpine) — stable channel

```sh
wget -O /etc/apk/keys/claude-code.rsa.pub \
  https://downloads.claude.ai/keys/claude-code.rsa.pub
echo "https://downloads.claude.ai/claude-code/apk/stable" >> /etc/apk/repositories
apk add claude-code

# Verify
sha256sum /etc/apk/keys/claude-code.rsa.pub
```

Switch to `latest`:
```sh
sed -i '\|downloads.claude.ai/claude-code/apk/stable|d' /etc/apk/repositories
echo "https://downloads.claude.ai/claude-code/apk/latest" >> /etc/apk/repositories
```

Upgrade: `apk update && apk upgrade claude-code`

## npm install

Requires **Node.js 18+**.

```bash
npm install -g @anthropic-ai/claude-code
```

Installs the same native binary as the standalone installer (via a per-platform optional dependency like `@anthropic-ai/claude-code-darwin-arm64`, then a postinstall link step). The installed `claude` does not invoke Node at runtime.

Supported platforms: `darwin-arm64`, `darwin-x64`, `linux-x64`, `linux-arm64`, `linux-x64-musl`, `linux-arm64-musl`, `win32-x64`, `win32-arm64`. Your package manager must allow optional dependencies.

**Upgrade**:
```bash
npm install -g @anthropic-ai/claude-code@latest
```

> **Don't use `npm update -g`** — it respects the semver range from the original install and may not move you to the newest release.

> **Don't use `sudo npm install -g`** — permission issues and security risks. If you hit permission errors, see the troubleshoot page.

## Binary integrity verification

Each release publishes `manifest.json` with per-platform SHA256 checksums. The manifest is signed with Anthropic's GPG key — verifying the signature transitively verifies every binary it lists.

> Manifest signatures available from **v2.1.89 onward**. Earlier releases publish checksums in `manifest.json` without a detached signature.

### Steps (POSIX shell; on Windows use Git Bash or WSL)

```bash
# 1. Import the release signing key
curl -fsSL https://downloads.claude.ai/keys/claude-code.asc | gpg --import
gpg --fingerprint security@anthropic.com
# Expected: 31DD DE24 DDFA B679 F42D  7BD2 BAA9 29FF 1A7E CACE

# 2. Download manifest + signature
REPO=https://downloads.claude.ai/claude-code-releases
VERSION=2.1.89
curl -fsSLO "$REPO/$VERSION/manifest.json"
curl -fsSLO "$REPO/$VERSION/manifest.json.sig"

# 3. Verify signature
gpg --verify manifest.json.sig manifest.json
# Expected: Good signature from "Anthropic Claude Code Release Signing <security@anthropic.com>"
# The "WARNING: This key is not certified..." line is normal for freshly imported keys.

# 4. Compare binary checksum
sha256sum claude          # Linux
shasum -a 256 claude      # macOS
# Or PowerShell: (Get-FileHash claude.exe -Algorithm SHA256).Hash.ToLower()
```

### Platform code signatures

In addition to the signed manifest:

| OS | Signed by | Verify with |
| --- | --- | --- |
| **macOS** | "Anthropic PBC" + Apple notarization | `codesign --verify --verbose ./claude` |
| **Windows** | "Anthropic, PBC" | `Get-AuthenticodeSignature .\claude.exe` |
| **Linux** | Not individually code-signed | Manifest signature above, or package-manager signature for apt/dnf/apk |

## Uninstall

### Binary removal by install method

| Install method | Uninstall |
| --- | --- |
| Native (macOS/Linux/WSL) | `rm -f ~/.local/bin/claude && rm -rf ~/.local/share/claude` |
| Native (Windows PowerShell) | `Remove-Item "$env:USERPROFILE\.local\bin\claude.exe" -Force; Remove-Item "$env:USERPROFILE\.local\share\claude" -Recurse -Force` |
| Homebrew | `brew uninstall --cask claude-code` (or `@latest`) |
| WinGet | `winget uninstall Anthropic.ClaudeCode` |
| apt | `sudo apt remove claude-code && sudo rm /etc/apt/sources.list.d/claude-code.list /etc/apt/keyrings/claude-code.asc` |
| dnf | `sudo dnf remove claude-code && sudo rm /etc/yum.repos.d/claude-code.repo` |
| apk | `apk del claude-code; sed -i '\|downloads.claude.ai/claude-code/apk|d' /etc/apk/repositories; rm /etc/apk/keys/claude-code.rsa.pub` |
| npm | `npm uninstall -g @anthropic-ai/claude-code` |

If `claude` still runs after uninstall: you have a second install or a leftover shell alias. Check the troubleshoot page for "Check for conflicting installations".

### Remove configuration files (optional, destructive)

> Removes all settings, allowed-tools, MCP configs, and session history.

```bash
# User scope
rm -rf ~/.claude
rm ~/.claude.json

# Project scope (run inside the project)
rm -rf .claude
rm -f .mcp.json
```

Windows PowerShell:
```powershell
Remove-Item "$env:USERPROFILE\.claude" -Recurse -Force
Remove-Item "$env:USERPROFILE\.claude.json" -Force
Remove-Item .claude -Recurse -Force
Remove-Item .mcp.json -Force
```

**Gotcha**: VS Code extension, JetBrains plugin, and Desktop app *also* write to `~/.claude/`. If any is still installed, the dir gets recreated next time they run. Uninstall those first if you want a complete wipe.

## Cross-OS playbook

### macOS host (this repo's primary)

Recommended: native installer. Auto-updates handle themselves. `claude doctor` once a week to confirm health. Channel: `latest` by default; switch to `stable` if you want to skip regressed releases (`/config` → Auto-update channel).

### Windows VM (VMware Fusion)

Recommended path:
1. PowerShell: `irm https://claude.ai/install.ps1 | iex`
2. Install Git for Windows (https://git-scm.com/downloads/win) **if you want POSIX-shell-friendly tooling** (Bash tool stays usable). Skip if you'd rather everything go through PowerShell.
3. Without Git for Windows, set `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` to make PowerShell the explicit shell tool.
4. `claude --version` and `claude doctor` to confirm.

Sandboxing isn't available on native Windows. If you need it (running risky scripts), switch to **WSL 2** — install the Linux installer inside the WSL terminal.

### Settings to consider on day 1

In `~/.claude/settings.json`:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "autoUpdatesChannel": "stable",
  "minimumVersion": "2.1.89"
}
```

`autoUpdatesChannel: "stable"` smooths over regressions. `minimumVersion` set to a known-good version prevents accidental downgrades when switching channels.

## Practical applications for this repo

- **For the Windows VM specifically**: copy the Quick install + Windows specifics sections into a `shared/project-setup/new-machine.md` when we get to writing that. The shell-identification table (`PS C:\` vs `C:\`) is the most common new-Windows-user paper cut.
- **For sandboxed development**: WSL 2 is the only Windows path that gets sandbox. If a client project needs sandboxed Bash, WSL 2 is non-negotiable.
- **Binary integrity** matters most for managed deployments; flag it in `shared/project-setup/` when we get there for security-conscious environments.
