# Permissions: rules + modes

How Claude Code decides whether to run a tool call. Distilled from `https://code.claude.com/docs/en/permissions` and `https://code.claude.com/docs/en/permission-modes`.

## Mental model

Two layers, evaluated together:

1. **Permission rules** (in `settings.json` under `permissions`) — fine-grained allow / ask / deny by tool and specifier.
2. **Permission modes** — session-level posture (`default`, `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`) that sets the baseline.

Tier behavior (default mode):

| Tool kind | Auto? | "Yes, don't ask again" lasts |
| --- | --- | --- |
| Read-only (Read, Grep, etc.) | Yes | n/a |
| Bash | Prompt | Permanently per project + command |
| File edits (Edit/Write) | Prompt | Until session end |

> **Rules are enforced by Claude Code, not the model.** `CLAUDE.md` and prompts shape *what Claude tries*; rules and hooks shape *what runs*. To enforce, use `/permissions`, settings rules, a mode change, or a `PreToolUse` hook.

## Evaluation order (the most-violated mental model)

Rules are evaluated **deny → ask → allow**. **First match wins regardless of specificity.**

- A broad deny like `Bash(aws *)` blocks every `aws` command, even ones a narrower allow rule like `Bash(aws s3 ls)` would otherwise permit. **Deny rules cannot carry allowlist exceptions.** Use a hook if you need that pattern.
- A matching ask rule still prompts even when a more specific allow rule also matches.

**Bare-name deny vs scoped deny:**

| Rule | Behavior |
| --- | --- |
| `Bash` (bare) | Tool removed from Claude's context entirely. Claude **never sees it**. |
| `Bash(rm *)` (scoped) | Tool stays available; matching calls blocked when attempted. |

This matters: `deny: ["WebFetch"]` makes Claude unaware WebFetch exists; `deny: ["WebFetch(domain:evil.com)"]` keeps WebFetch usable for everything else.

## Permission rule syntax

### Form

```
Tool                   # match all uses
Tool(specifier)        # match specific uses
Tool(param:value)      # match by input parameter (deny/ask only)
```

`Bash(*)` is equivalent to bare `Bash`. As deny, both forms remove the tool from context.

### Match by input parameter (deny/ask only)

Match a top-level input field on any tool with `Tool(param:value)`. Only for deny/ask — allow rules would be unsafe because they'd suggest the whole call is safe, which isn't true.

| Rule | Matches |
| --- | --- |
| `Agent(model:opus)` | Agent calls requesting Opus model tier |
| `Agent(isolation:worktree)` | Agent calls requesting a git worktree |
| `Bash(run_in_background:true)` | Bash calls running in background |

Rules:
- Direct field only, not nested.
- One param per rule (combine via two rules).
- `*` wildcard supported; without it, exact match.
- Omitted param never matches (so `Agent(model:*)` won't match a call that leaves `model` unset).
- Compared against literal input before normalization (so `Agent(model:opus)` matches the alias but not the full ID).
- Whitespace around `:` ignored.

Fields that have their own canonicalizing syntax can't be matched this way and emit a startup warning if you try: `command` (Bash/PowerShell), `file_path` (Read/Edit/Write), `path` (Grep/Glob), `notebook_path` (NotebookEdit), `url` (WebFetch). Use the canonical form: `Bash(rm *)`, `Read(./path)`, `WebFetch(domain:host)`.

### Tool-name wildcards

| In rule type | Allowed |
| --- | --- |
| Deny / ask | `*` (all tools), `Bash*`, `mcp__*` (all MCP) |
| Allow | Only after literal `mcp__<server>__` prefix, server segment glob-free (e.g. `mcp__github__get_*`). Other allow globs ignored with a warning. |

Typos in tool names emit a startup warning (except names containing `_` or `*`). **Use canonical names from `tools-reference`** — the label shown in transcripts and dialogs can differ (e.g. transcript "Stop Task" → canonical `TaskStop`).

## Bash rules in detail

Wildcards anywhere; `:*` is the same as ` *` at the trailing position only.

### Word boundaries

The space matters:

| Rule | Matches `ls -la` | Matches `lsof` |
| --- | --- | --- |
| `Bash(ls *)` | Yes (space-anchored, needs space or end) | No |
| `Bash(ls*)` | Yes | Yes |

The permission UI writes the space-separated form when you click "Yes, don't ask again" for a prefix.

### Compound commands

Recognized separators: `&&`, `||`, `;`, `|`, `|&`, `&`, newlines.

- Each subcommand must match independently.
- Approving a compound saves a **separate rule per subcommand** (up to 5).
- `cd subdir` in a compound generates its own Read rule for the path.

### Process wrappers (auto-stripped before matching)

Stripped: `timeout`, `time`, `nice`, `nohup`, `stdbuf`. So `Bash(npm test *)` matches `timeout 30 npm test`.

`xargs` is stripped **only when it has no flags**. `xargs -n1 grep pattern` is treated as an `xargs` command, not a `grep` command.

**Not stripped** — these execute their arguments as a command, so a permissive rule on the runner is a hole: `direnv exec`, `devbox run`, `mise exec`, `npx`, `docker exec`. Write narrow rules like `Bash(devbox run npm test)`, not `Bash(devbox run *)`.

**Always prompt** (can't be auto-approved by prefix rule): `watch`, `setsid`, `ionice`, `flock`, and `find` with `-exec` or `-delete`. Write exact-match rules.

### Built-in read-only commands (auto-allowed in every mode)

`ls`, `cat`, `echo`, `pwd`, `head`, `tail`, `grep`, `find`, `wc`, `which`, `diff`, `stat`, `du`, `cd`, and read-only forms of `git`. Not configurable — add an `ask` or `deny` rule to require a prompt for one of these.

Unquoted globs allowed for these (since every flag is read-only). `cd` into a path inside your working dir or an additional dir is read-only. **`cd` combined with `git` in one compound always prompts**, regardless of target.

### Bash URL filtering is fragile

`Bash(curl http://github.com/ *)` is bypassable by query strings, redirects, env-var-built URLs, etc. Use the WebFetch tool with `WebFetch(domain:...)` allow rules and **deny `curl` / `wget` in Bash**. Or a `PreToolUse` hook that validates URLs.

## PowerShell rules

Same shape as Bash. Differences:

- AST is parsed; pipeline `|`, statement `;`, and (PS7+) `&&` / `||` split into subcommands evaluated independently.
- **Common aliases canonicalized** — `PowerShell(Get-ChildItem *)` matches `gci`, `ls`, `dir`. Case-insensitive.

To use PowerShell as the shell on Windows: set `defaultShell: "powershell"` and `CLAUDE_CODE_USE_POWERSHELL_TOOL=1` (see `settings.md`).

## Read and Edit rules — the anchor traps

`Edit` covers all built-in file-editing tools; `Read` is best-effort across Read, Grep, Glob, `@file` mentions, and IDE-shared context.

**Gitignore semantics with four anchor types:**

| Anchor | Means | Example | Resolves to |
| --- | --- | --- | --- |
| `//path` | **Absolute** from filesystem root | `Read(//Users/alice/secrets/**)` | `/Users/alice/secrets/**` |
| `~/path` | Home-relative | `Read(~/Documents/*.pdf)` | `$HOME/Documents/*.pdf` |
| `/path` | **Project-root-relative** (not absolute!) | `Edit(/src/**/*.ts)` | `<project>/src/**/*.ts` |
| `path` or `./path` | CWD-relative | `Read(*.env)` | `<cwd>/*.env` |

> **Gotcha:** `/Users/alice/file` is **not** an absolute path — it's project-relative. Use `//Users/alice/file` for absolute. Sandbox filesystem paths use the opposite convention (`/` is absolute). Easy place to make a mistake.

### Windows path normalization

POSIX form before matching. `C:\Users\alice` → `/c/Users/alice`. So:
- `//c/**/.env` matches `.env` anywhere on C: drive.
- `//**/.env` matches across all drives.

### Bare filenames behave like gitignore

`Read(.env)` = `Read(**/.env)` — matches at any depth under the CWD. To match anywhere on the filesystem use the absolute anchor: `Read(//**/.env)`.

### Symlinks

Two paths checked: the symlink and its target. Different treatment:

- **Allow rules**: apply only when **both** match. A symlink inside an allowed dir pointing outside still prompts.
- **Deny rules**: apply when **either** matches. A symlink to a denied file is denied.

### Read/Edit limits

Apply to Claude's built-in file tools and to file-related Bash commands Claude Code recognizes (`cat`, `head`, `tail`, `sed`). They **do not apply to arbitrary subprocesses** (e.g. a Python script that opens files itself). For OS-level enforcement, use the sandbox.

## WebFetch rules

`WebFetch(domain:...)` matches the requested hostname, case-insensitive.

| Rule | Effect |
| --- | --- |
| `WebFetch(domain:example.com)` | Exact host `example.com` |
| `WebFetch(domain:*.example.com)` | Any subdomain (`api.example.com`, `a.b.example.com`) but **not** `example.com` itself |
| `WebFetch(domain:*)` | All domains (= bare `WebFetch`) |

Mid-position wildcards match only **between dots** — `example.*` matches `example.org` but **not** `example.evil.com`. This is what makes the rule resistant to attacker-registered domains.

> Pair WebFetch allow rules with Bash deny rules for `curl`/`wget` — WebFetch alone doesn't prevent Bash from reaching any URL.

## MCP rules

```
mcp__puppeteer                              # any tool from puppeteer server
mcp__puppeteer__*                           # equivalent
mcp__puppeteer__puppeteer_navigate          # specific tool
mcp__github__get_*                          # all get_ tools from github server (allow ok)
mcp__*                                      # all MCP tools — deny/ask only
```

## Agent (subagents) rules

```
Agent(Explore)                              # built-in Explore subagent
Agent(Plan)                                 # built-in Plan subagent
Agent(my-custom-agent)                      # custom subagent
```

`Agent` allow rules are **dropped on entering auto mode** (alongside broad Bash allow rules) and restored on leaving.

## Cd rules (the `/cd` command)

`/cd` is user-driven only — Claude can't invoke it. Path patterns share Read/Edit anchors but are anchored to the whole directory path (not gitignore-style). `*` matches one segment; `**` across segments. A trailing `/**` also matches its named root.

| Rule | Matches | Doesn't match |
| --- | --- | --- |
| `Cd(~/code/*)` | `~/code/app` | `~/code/app/src`, `~/code` |
| `Cd(~/code/**)` | `~/code` and anything under | dirs outside `~/code` |
| `Cd(**/node_modules)` | any `node_modules` at any depth | `node_modules/pkg` |

- Bare `Cd` deny disables `/cd` entirely.
- Any `Cd` allow rule → allowlist mode (target must match an allow).
- No `Cd` rules → default behavior (trust-prompt for unknown).
- Deny rules check every spelling including symlink hops.

## Permission modes (the session-level posture)

Six modes. The mode sets the baseline; rules layer on top.

| Mode | Auto-runs | Best for |
| --- | --- | --- |
| `default` | Reads only | Sensitive work, getting started |
| `acceptEdits` | Reads + edits + common fs commands | Iterating on code you'll review later |
| `plan` | Reads only; plans without editing | Exploring before changing |
| `auto` | Everything, with background classifier checks | Long tasks; reducing prompt fatigue |
| `dontAsk` | Only pre-approved tools | Locked-down CI/scripts |
| `bypassPermissions` | Everything | Isolated containers/VMs only |

In every mode except `bypassPermissions`, writes to **protected paths** (see below) are never auto-approved.

### `acceptEdits`

Auto-approves on paths inside the working directory or `additionalDirectories`:

- File edits (Edit/Write).
- Common fs Bash commands: `mkdir`, `touch`, `rm`, `rmdir`, `mv`, `cp`, `sed`.
- Safe-env-prefixed variants: `LANG=C ...`, `NO_COLOR=1 ...`.
- Process-wrapped variants: `timeout ...`, `nice ...`, `nohup ...`.
- With PowerShell tool enabled: `Set-Content`, `Add-Content`, `Clear-Content`, `Remove-Item` and their aliases.

Outside that scope, protected paths, and all other Bash → still prompt.

### `plan`

Reads files + runs read-only shell commands; doesn't edit. Enter via `Shift+Tab` cycle, `/plan` prefix, or `--permission-mode plan`.

When the plan is ready, Claude presents it and asks how to proceed:

- Approve and start in **auto** mode
- Approve and **acceptEdits**
- Approve and **review each edit manually**
- **Keep planning** with feedback
- Refine with **Ultraplan** (browser-based review)

Approving switches the session to whichever mode you picked. `Ctrl+G` opens the proposed plan in your default text editor for direct edits before Claude proceeds. With `showClearContextOnPlanAccept: true`, each option also offers to clear planning context first. Accepting a plan auto-names the session from the content.

Set `defaultMode: "plan"` to make plan-by-default for a project.

### `auto` (research preview, v2.1.83+)

Claude executes without routine prompts; a separate **classifier model** vets each action before it runs. Explicit ask rules still prompt; explicit deny rules still block.

> **Auto mode is a research preview.** Reduces prompts, doesn't guarantee safety. Use where you trust the direction.

**Requirements:**
- Available on all plans; on Team/Enterprise, admin must enable in `claude.ai/admin-settings/claude-code`.
- Models: Anthropic API needs Opus 4.6+ or Sonnet 4.6. **Bedrock / Vertex / Foundry: only Opus 4.7 / 4.8.**
- On Bedrock / Vertex / Foundry, also set `CLAUDE_CODE_ENABLE_AUTO_MODE=1` in `env` (v2.1.158+).

**Repo-spoof protection:** `defaultMode: "auto"` is **ignored from `.claude/settings.json` and `.claude/settings.local.json`** (v2.1.142+). Set it in `~/.claude/settings.json` to make auto your personal default.

**Classifier blocks by default:**

- `curl | bash` and other code-fetch-and-exec.
- Sending sensitive data to external endpoints.
- Production deploys / migrations.
- Mass cloud-storage deletion.
- Granting IAM or repo permissions.
- Modifying shared infrastructure.
- Irreversibly destroying files that existed before the session.
- Force push; pushing directly to `main`.
- **Git rewriters that discard work:** `git reset --hard`, `git checkout -- .`, `git restore .`, `git clean -fd`, `git stash drop`, `git stash clear` (v2.1.182+).
- `git commit --amend` when HEAD wasn't created in this session.
- `terraform/pulumi/cdk/terragrunt destroy`; applying a plan that destroys resources.

**Classifier allows by default:**

- Local file ops in the working dir.
- Installing deps declared in lock files / manifests.
- Reading `.env` and sending creds to their matching API.
- Read-only HTTP requests.
- Pushing to the branch you started on or one Claude created.

**Configure trusted infrastructure** via `autoMode.environment` if routine actions get blocked.

**Boundaries stated in chat** ("don't push", "wait until I review") are honored as block signals — but they're re-read from the transcript each check, so they're **lost when context compaction removes the stating message**. For hard guarantees, use deny rules.

**Broad allow rules dropped on entering auto:** `Bash(*)`, `Bash(python*)`, package-manager run commands, `Agent` allow rules. Narrow rules like `Bash(npm test)` carry over. Restored on leaving auto.

**Subagents** (v2.1.178+): classifier vets the delegated task at spawn, each action during the run, and reviews the full action history on return.

**Fallbacks:**

- Each denial appears in `/permissions` → Recently denied tab; press `r` to retry with manual approval.
- After **3 denials in a row** or **20 total**, auto mode pauses and prompts. Approving resumes auto. Counters: consecutive resets on any allow; total persists until its own threshold fires.
- In `-p` (headless), repeated blocks abort the session.

Classifier runs on a server-configured model independent of your `/model`. **Counts toward token usage** and adds a round-trip on each shell/network operation (reads + working-dir edits skip it).

### `dontAsk`

Auto-deny everything that would prompt. Only `permissions.allow` rules and the built-in read-only Bash set execute. Explicit ask rules → denied (instead of prompting). Fully non-interactive — for CI/scripts.

Cloud sessions on Claude Code for the web **ignore `defaultMode: "dontAsk"`**.

Set with `--permission-mode dontAsk`.

### `bypassPermissions`

Skip every check. **Includes writes to protected paths as of v2.1.126** (earlier versions still prompted).

Still prompts:
- Explicit ask rules.
- `rm -rf /` and `rm -rf ~` (circuit breaker against model error).

**Won't enter from a session that didn't start with one of the enabling flags** — restart with `--permission-mode bypassPermissions` or `--dangerously-skip-permissions` (equivalent).

**Refuses to start as root/sudo** on Linux/macOS unless inside a recognized sandbox. For autonomous runs, use the official dev container which runs as a non-root user.

Cloud sessions also ignore `defaultMode: "bypassPermissions"`.

Lock off org-wide via `permissions.disableBypassPermissionsMode: "disable"` in managed settings.

> Provides **no protection against prompt injection or unintended actions.** For background safety checks with far fewer prompts, use auto mode.

## Switching modes

| Where | How |
| --- | --- |
| CLI mid-session | `Shift+Tab` cycles `default → acceptEdits → plan` |
| CLI startup | `--permission-mode <mode>` (also works with `-p`) |
| Settings default | `permissions.defaultMode` |
| VS Code | Mode indicator at bottom of prompt box; or `claudeCode.initialPermissionMode` (doesn't accept `auto` — use `defaultMode` for that) |
| JetBrains | Same as CLI (runs in IDE terminal) |
| Desktop | Mode selector next to send button |
| Web/mobile | Mode dropdown next to prompt; cloud sessions limited to Accept edits / Plan / Auto |

Optional modes slot into the `Shift+Tab` cycle after `plan` only when explicitly enabled (`auto` requires opt-in prompt; `bypassPermissions` requires startup with enabling flag; `dontAsk` never in cycle, flag-only).

## Protected paths

Writes to these are never auto-approved in any mode **except** `bypassPermissions`:

| Mode | Protected writes |
| --- | --- |
| `default`, `acceptEdits`, `plan` | Prompted |
| `auto` | Routed to classifier |
| `dontAsk` | Denied |
| `bypassPermissions` | Allowed |

**`permissions.allow` rules don't pre-approve protected-path writes** — the safety check runs before allow rules. In modes that prompt, the prompt offers "Yes, and allow Claude to edit its own settings for this session" which approves further `.claude/` writes in that session.

**Protected directories:** `.git`, `.config/git`, `.vscode`, `.idea`, `.husky`, `.cargo`, `.devcontainer`, `.yarn`, `.mvn`, `.claude` (except `.claude/worktrees`).

**Protected files:**
- Git: `.gitconfig`, `.gitmodules`.
- Shells / env: `.bashrc`, `.bash_profile`, `.bash_login`, `.bash_aliases`, `.bash_logout`, `.zshrc`, `.zprofile`, `.zshenv`, `.zlogin`, `.zlogout`, `.profile`, `.envrc`.
- Package managers: `.npmrc`, `.yarnrc`, `.yarnrc.yml`, `.pnp.cjs`, `.pnp.loader.mjs`, `.pnpmfile.cjs`, `bunfig.toml`, `.bunfig.toml`.
- Build / tooling: `.bazelrc`, `.bazelversion`, `.bazeliskrc`, `gradle-wrapper.properties`, `maven-wrapper.properties`.
- Hooks: `.pre-commit-config.yaml`, `lefthook.yml`/`.lefthook.yaml`/etc.
- Editor / lint: `.ripgreprc`, `pyrightconfig.json`, `.devcontainer.json`.
- Claude: `.mcp.json`, `.claude.json`.

## Working directories

By default Claude has access to the launch directory only. Extend with:

| Mechanism | Persistence | What it grants |
| --- | --- | --- |
| `--add-dir <path>` | One session | File access **+** loads skills, subagents, plugin keys, optionally CLAUDE.md (env-var-gated). Live reload for skills. |
| `/add-dir` | One session | Same as `--add-dir`. |
| `permissions.additionalDirectories` | Permanent (per scope) | **File access only.** No config loading from these dirs. |
| `/cd <path>` (v2.1.169+) | Until next `/cd` | Relocates the session: loads new `CLAUDE.md`, `--resume` finds session from there. |

**Key distinction**: `--add-dir`/`/add-dir` *and* `additionalDirectories` both grant file access — but only the flag/command versions also load configuration. For configuration shared across projects, prefer:
- User-level (`~/.claude/agents/`, `~/.claude/settings.json`),
- Plugins, or
- Launching from the config directory directly.

## Hooks integration

`PreToolUse` hooks register custom logic that runs before the permission prompt. Hook output can `deny`, force `ask`, or `allow`.

**Hooks don't bypass rules:**
- Deny rules and ask rules apply regardless of hook output.
- A matching deny still blocks even when the hook returned `"allow"`.
- A matching ask still prompts even when the hook returned `"allow"`.
- A hook that exits with code 2 blocks before rules are evaluated — overrides allow rules.

**Use case**: bare `Bash` allow + a `PreToolUse` hook that rejects specific commands = run everything without prompts except the few you block.

## Sandbox interaction

Permissions and sandboxing are complementary:

- **Permissions** apply to all tools.
- **Sandbox** is OS-level, **Bash-only**, restricts the filesystem and network reach of subprocesses.

Combined behaviors:

- `sandbox.filesystem.allowWrite/denyWrite/denyRead/allowRead` merge with `Edit(...)` and `Read(...)` permission paths into the sandbox boundary.
- `sandbox.network.allowedDomains` / `deniedDomains` merge with `WebFetch(domain:...)` permission rules.
- With `autoAllowBashIfSandboxed: true` (default), sandboxed Bash skips the bare `Bash` ask rule prompt — the sandbox boundary substitutes. **Content-scoped ask rules like `Bash(git push *)` still prompt.** `rm`/`rmdir` targeting `/`, home, or critical system paths still prompt.
- Commands in `sandbox.excludedCommands` respect the bare `Bash` ask rule normally.

## Managed-only settings (org admins)

| Setting | Effect |
| --- | --- |
| `allowManagedPermissionRulesOnly` | User/project can't define allow/ask/deny. Only managed rules apply. |
| `allowManagedMcpServersOnly` | Only managed `allowedMcpServers` respected (deny still merges). |
| `allowManagedHooksOnly` | Only managed/SDK/managed-plugin hooks load. |
| `sandbox.filesystem.allowManagedReadPathsOnly` | Only managed `allowRead` paths honored. |
| `sandbox.network.allowManagedDomainsOnly` | Only managed `allowedDomains` honored; others auto-blocked. |
| `disableBypassPermissionsMode` / `disableAutoMode` | Lock off `bypassPermissions` / `auto`. |
| `strictPluginOnlyCustomization` | Lock skills/agents/hooks/mcp to plugins + managed. |
| `wslInheritsWindowsSettings` | WSL reads Windows policy chain too. |

## Precedence (recap)

Same as global settings:

1. **Managed**
2. **CLI args** (`--allowedTools`, `--disallowedTools`, `--permission-mode`)
3. **Local project** (`.claude/settings.local.json`)
4. **Shared project** (`.claude/settings.json`)
5. **User** (`~/.claude/settings.json`)

**Array merging**: `permissions.allow / ask / deny` and `additionalDirectories` merge + dedupe across scopes. If a tool is denied at any level, no other level can allow it.

## Cross-OS notes

- **Read/Edit anchors**: `//path` absolute, `/path` project-relative — same on macOS and Windows.
- **Windows paths normalize to POSIX**: `C:\Users\alice\.env` → `/c/Users/alice/.env`. Use `//c/**/.env` to match `.env` anywhere on C:.
- **PowerShell rules**: aliases canonicalized — `PowerShell(Get-ChildItem *)` matches `gci`, `ls`, `dir`. Only relevant when `defaultShell: "powershell"`.
- **Sandbox**: macOS Seatbelt, Linux/WSL2 bubblewrap; **no native Windows sandbox**, so permissions are the only Bash gate on native Windows. Compensate with stricter `permissions.deny` there.
- **`bypassPermissions` refuses root/sudo on Linux/macOS** unless inside a recognized sandbox. On the Windows VM there's no equivalent root check — be more careful.

## Practical recipes

### Solo dev defaults — checked in (`.claude/settings.json`)

```json
{
  "permissions": {
    "defaultMode": "default",
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)",
      "Bash(curl http*)",
      "Bash(wget *)",
      "Bash(git push * main)",
      "Bash(git push * master)",
      "Bash(git push --force *)"
    ],
    "ask": [
      "Bash(rm -rf *)",
      "Bash(npm publish *)"
    ]
  }
}
```

### Personal overrides (`.claude/settings.local.json`)

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(npm test *)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(git status)",
      "Bash(git add *)",
      "Bash(git commit *)"
    ]
  }
}
```

### Personal default mode (`~/.claude/settings.json`)

```json
{
  "permissions": {
    "defaultMode": "plan"
  }
}
```

`plan` as personal default = every session starts by exploring before editing. Toggle to `acceptEdits` with `Shift+Tab` once you're sure of direction.

### Hard "never touch these" (`~/.claude/settings.json`)

```json
{
  "permissions": {
    "deny": [
      "Read(~/.ssh/**)",
      "Read(~/.aws/credentials)",
      "Read(~/.gnupg/**)",
      "Edit(~/.bashrc)",
      "Edit(~/.zshrc)"
    ]
  }
}
```

(Dotfiles in `~/` aren't all protected paths automatically — `.bashrc` etc. are protected from writes but not from reads. Add deny rules for reads.)

## Practical applications for this repo

- The current state of `permissions` is empty everywhere — fine for a docs repo. Once we start drafting hooks/scripts in `claude/config/` we'll want at least the `deny` block above checked in.
- The Windows VM will lack sandbox protection — when we add Windows-specific examples to the playbook, suggest stricter `permissions.deny` rules there to compensate.
- Auto mode is worth trying eventually — but the repo-spoof rule means we cannot make this repo auto-mode-default by checking it into `.claude/settings.json`. Personal default → `~/.claude/settings.json`.
