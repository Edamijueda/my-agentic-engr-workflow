---
verified_against: 2.1.308
verified_on: 2026-10-31
sources:
  - https://code.claude.com/docs/en/permissions
  - https://code.claude.com/docs/en/permission-modes
---

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

Match a top-level input field on any **built-in** tool with `Tool(param:value)`. Only for deny/ask — allow rules would be unsafe because they'd suggest the whole call is safe, which isn't true.

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
| Deny / ask | `*` (all tools), `Bash*`, `mcp__*` (all MCP). Must match the **full** tool name. A bare-name glob deny removes matched tools from context, same as a bare tool name — except `EndConversation`, which a glob can't remove while any other tool remains. |
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

- For an **allow** to cover a compound, every subcommand must match independently (or be a read-only command).
- **Deny and ask rules fire if *any* subcommand matches** — including one nested in a subshell, a command substitution, or a control-flow body. An ask rule `Bash(git clean *)` still prompts for `cd /tmp && git clean -f` and for `echo "$(git clean -f)"`, **even in auto mode**.
- A trailing `&&` or `||` with nothing after it makes the command unparseable, so it isn't split for allow matching — `Bash(npm *)` does not approve `npm test &&`.
- Approving a compound saves a **separate rule per subcommand** (up to 5).
- `cd` into a directory *outside* your working directories generates its own Read rule for that path.

### Process wrappers (auto-stripped before matching)

Stripped: `timeout`, `time`, `nice`, `nohup`, `stdbuf`, the shell builtins `command` and `builtin`, and zsh's `noglob`. So `Bash(npm test *)` matches `timeout 30 npm test`. **Not** stripped: the query form `command -v` (it looks a command up rather than running it) and zsh's `nocorrect`. The list is built in and not configurable.

`xargs` is stripped **only when it has no flags**. `xargs -n1 grep pattern` is treated as an `xargs` command, not a `grep` command.

**Leading variable assignments cut differently for allow than for deny.** Claude Code strips a leading assignment of certain known-safe variables, so `Bash(npm test *)` matches `NODE_ENV=test npm test`. An **allow** rule won't match past an assignment of any *other* variable. A **deny or ask** rule matches past any leading assignment — `Bash(rm *)` in deny still catches `FOO=bar rm -rf tmp/`.

**Not stripped** — these execute their arguments as a command, so a permissive rule on the runner is a hole: `direnv exec`, `devbox run`, `mise exec`, `npx`, `docker exec`. Write narrow rules like `Bash(devbox run npm test)`, not `Bash(devbox run *)`.

**Always prompt** (can't be auto-approved by prefix rule): `watch`, `setsid`, `ionice`, `flock`, and `find` with `-exec` or `-delete`. Write exact-match rules.

### Built-in read-only commands (auto-allowed in every mode)

`ls`, `cat`, `echo`, `pwd`, `head`, `tail`, `grep`, `find`, `wc`, `which`, `diff`, `stat`, `du`, `cd`, and read-only forms of `git`. Not configurable — add an `ask` or `deny` rule to require a prompt for one of these.

Unquoted globs are allowed **only for commands whose every flag is read-only** — `ls *.ts`, `wc -l src/*.py`. Commands with write- or exec-capable flags (`find`, `sort`, `sed`, `git`) prompt when an unquoted glob is present, because the glob could expand into a flag like `-delete`.

A `cd` into a path inside your working dir or an additional dir is read-only, and `cd packages/api && ls` runs unprompted when each part qualifies. Two combinations still prompt:

- **`cd` with `git`** — but only when the `cd` actually changes directory. A `cd` that resolves to the current working directory is a no-op and doesn't prompt. (Running `git` in a new directory can execute that directory's hooks.)
- **`cd` with a redirect**, when Claude Code can't tell what the redirect target resolves against after the `cd`. A redirect to `/dev/null` doesn't prompt, since it doesn't depend on the working directory.

A redirect on its own adds a check on the target: `ls > out.txt` is not treated as read-only.

**Other cases where a read-only command still prompts in Manual mode:** `docker` carrying a flag that selects a different daemon (`-H`, `--context`, Podman's `--url`/`--connection`); `file` with `-m`/`--magic-file` or `-f`/`--files-from` (both open paths named in the value); `ps` where an argument could act as the `e` option (`ps auxe`, `ps aux -e` — but `ps aux` and `ps -ef` are fine); a command that sets, unsets or loops over a special shell variable such as `PATH` or `IFS`; anything Claude Code can't fully parse; and **any command over 10,000 characters**, which always prompts.

### What a Bash rule doesn't match

A rule matches the command text Claude writes, after compound-splitting and wrapper-stripping. It is **not** a boundary around the program:

| Rule | Stops | Doesn't stop |
| --- | --- | --- |
| `Bash(curl *)` | `curl https://example.com` | `/usr/bin/curl ...`, `sh -c 'curl ...'` |
| `Bash(rm *)` | `rm -rf build/` | `/bin/rm -rf build/`, `bash -c 'rm -rf build/'` |
| `Bash(git push *)` | `git push origin main` | `git -C . push origin main`, `git -c push.default=current push origin main`, `git 'push' origin main` |

Your other rules and the permission mode decide the right-hand column. For enforcement that doesn't depend on command text, use the sandbox or a `PreToolUse` hook.

### Bash URL filtering is fragile

`Bash(curl http://github.com/ *)` is bypassable by query strings, redirects, env-var-built URLs, etc. Use the WebFetch tool with `WebFetch(domain:...)` allow rules and **deny `curl` / `wget` in Bash**. Or a `PreToolUse` hook that validates URLs.

## PowerShell rules

Same shape as Bash. Differences:

- AST is parsed; pipeline `|`, statement `;`, and (PS7+) `&&` / `||` split into subcommands evaluated independently.
- **Common aliases canonicalized** — `PowerShell(Get-ChildItem *)` matches `gci`, `ls`, `dir`. Case-insensitive.

Only relevant if you've enabled the PowerShell tool — see [`../tools/tools-reference.md`](../tools/tools-reference.md).

## Read and Edit rules — the anchor traps

`Edit` covers all built-in file-editing tools. `Read` is best-effort across the built-in tools that read files (Read, and `Grep`/`Glob` where they're loaded), `@file` mentions in your prompts, and the selection and open-file context a connected IDE shares.

> On macOS, Linux and WSL, `Grep` and `Glob` usually aren't loaded — so the part of that coverage you actually get is the Read tool, `@file` mentions and IDE context. See "Read/Edit limits" below for what reaches Bash instead.

**Gitignore semantics with four anchor types:**

| Anchor | Means | Example | Resolves to |
| --- | --- | --- | --- |
| `//path` | **Absolute** from filesystem root | `Read(//Users/alice/secrets/**)` | `/Users/alice/secrets/**` |
| `~/path` | Home-relative | `Read(~/Documents/*.pdf)` | `$HOME/Documents/*.pdf` |
| `/path` | **Project-root-relative** (not absolute!) | `Edit(/src/**/*.ts)` | `<project>/src/**/*.ts` |
| `path` or `./path` | CWD-relative | `Read(*.env)` | `<cwd>/*.env` |

> **Gotcha:** `/Users/alice/file` is **not** an absolute path — it's project-relative. Use `//Users/alice/file` for absolute. Sandbox filesystem paths use the opposite convention (`/` is absolute). Easy place to make a mistake.

### Bare filenames behave like gitignore

`Read(.env)` = `Read(**/.env)` — matches at any depth under the CWD. To match anywhere on the filesystem use the absolute anchor: `Read(//**/.env)`.

### Carve-outs can't reopen a blocked directory

With `Read(secrets/**)` denying and `Read(!secrets/public/**)` carving out, `secrets/public` stays blocked along with the rest — a negation can't reopen a file inside a directory a rule blocks as a whole.

### Network-path reads get their own check

A read from a path that can reach another machine — a UNC path on Windows, a `/net` automount path on macOS and Linux — is checked separately, because resolving it contacts the host. In auto mode the prompt comes to you and the classifier doesn't decide it.

### Symlinks

Two paths checked: the symlink and its target. Different treatment:

- **Allow rules**: apply only when **both** match. A symlink inside an allowed dir pointing outside still prompts.
- **Deny rules**: apply when **either** matches. A symlink to a denied file is denied.

### Read/Edit limits

**Covered:**

- Claude's built-in file tools.
- File commands Claude Code recognizes in Bash when they **name the file** — `cat`, `head`, `tail`, `sed`, `tee`.
- The targets of Bash **redirections**: `> file` checks `Edit` rules, `< file` checks `Read` rules (v2.1.257+). A `tee` destination is checked against `Edit` rules, protected paths and the working directories (v2.1.269+) — and a `Bash(tee *)` allow rule does **not** cover a destination outside them.

**Not covered:**

- **A command that reads files without naming them** — the documented example is `grep -r pattern .` run from the directory holding the file. This matters more than it looks on macOS, where [`Glob` and `Grep` aren't in the default tool set](../tools/tools-reference.md) and search *is* `grep` through Bash.
- Arbitrary subprocesses that open files themselves — a Python or Node script.

> **So `Read` denies are not a secrets boundary.** `Read(./.env)` stops the Read tool, an `@file` mention, IDE-shared context and `cat .env`. It does not stop `grep -r AWS_SECRET .`. For a guarantee, escalate: a `PreToolUse` hook that inspects the command, or `sandbox.filesystem.denyRead`, which is OS-level and covers every subprocess. See [`../../shared/principles/enforcement-vs-influence.md`](../../shared/principles/enforcement-vs-influence.md).

## WebFetch rules

`WebFetch(domain:...)` matches the requested hostname, case-insensitive, with a trailing `.` stripped from both rule and hostname (`example.com.` ≡ `example.com`). Wildcard matching needs v2.1.172+.

| Rule | Effect |
| --- | --- |
| `WebFetch(domain:example.com)` | Exact host `example.com` |
| `WebFetch(domain:*.example.com)` | Any subdomain (`api.example.com`, `a.b.example.com`) but **not** `example.com` itself |
| `WebFetch(domain:*)` | All domains — **not** the same as a bare `WebFetch` rule. Both cover every URL, but only the `domain:` form also feeds the sandbox's allowed/denied domain list. |

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
| `default` (CLI alias `manual`) | Reads only | Sensitive work, getting started |
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
- Safe-env-prefixed (`LANG=C ...`, `NO_COLOR=1 ...`) and process-wrapped (`timeout`, `nice`, `nohup`) variants of those — **except `touch`**, which only auto-approves as a bare command on plain file paths. A `touch` with an option, a glob, or any prefix still prompts.
- With PowerShell tool enabled: `Set-Content`, `Add-Content`, `Clear-Content`, `Remove-Item` and their aliases. A *positional* argument containing a quote character still prompts — pass content through a named parameter such as `-Value`.

Still prompts: paths outside the working directories, protected-path writes, recognized `rm`/`rmdir` critical-path removals, and every other Bash command except the built-in read-only set. **Symlinks that resolve outside the working directory are not auto-approved.**

### `plan`

Claude reads, explores with shell commands, and writes a plan without editing source. Enter via `Shift+Tab` cycle, `/plan` prefix, or `--permission-mode plan`. `Shift+Tab` again exits without approving.

**Which shell commands run while planning** depends on the session:

| Situation | Behaviour |
| --- | --- |
| Interactive terminal with bypass available | No classifier and no prompt applies to planning commands — and plan mode's edit blocks aren't enforced either |
| Auto available, `useAutoModeDuringPlan` on (default) | Classifier reviews shell commands other than critical-path removals |
| Otherwise | Anything outside the built-in read-only set prompts, even with sandbox auto-allow on |

When the plan is ready, Claude presents it and asks how to proceed:

- Approve and start in **auto** mode
- Approve and **acceptEdits**
- Approve and **review each edit manually**
- **Keep planning** with feedback
- Refine with **Ultraplan** (browser-based review)

Approving switches the session to whichever mode you picked. `Ctrl+G` opens the proposed plan in your default text editor for direct edits before Claude proceeds. With `showClearContextOnPlanAccept: true`, each option also offers to clear planning context first. Accepting a plan auto-names the session from the content.

Set `defaultMode: "plan"` to make plan-by-default for a project.

### `auto` (default since v2.1.283)

Claude executes without routine prompts; a separate **classifier model** vets each action before it runs. Explicit ask rules still prompt; explicit deny rules still block.

**This is now the starting mode** for interactive terminal and VS Code sessions — you are in it unless you changed it. It was a research preview when this playbook first covered it; that framing is gone, but the caveat isn't: auto mode reduces prompts, it does not guarantee safety.

**Requirements:**
- All plans. On Team/Enterprise it's on by default — admins turn it *off* with `permissions.disableAutoMode: "disable"` in managed settings.
- Models, by provider:

  | Provider | Supported |
  | --- | --- |
  | Anthropic API, Claude Platform on AWS | Opus 4.6+, Sonnet 4.6+, Haiku 5.5, any Fable |
  | Bedrock, Google Cloud's Agent Platform, Foundry, signed-in apps gateway | Sonnet 5+, Opus 4.7+, Haiku 5.5, any Fable |

  Sonnet 4.5, Opus 4.5, Haiku 4.5 and claude-3 models are unsupported everywhere. Haiku 5.5 on the four cloud providers needs v2.1.293+.
- No env var required. `CLAUDE_CODE_ENABLE_AUTO_MODE=1` is no longer needed on the cloud providers.
- Optional knobs: `CLAUDE_CODE_AUTO_MODE_MODEL` picks the classifier model (v2.1.302+); `CLAUDE_CODE_AUTO_MODE_SERVER` controls whether the server runs the classifier review.

**Repo-spoof protection:** `defaultMode: "auto"` is **ignored from `.claude/settings.json` and `.claude/settings.local.json`** . Set it in `~/.claude/settings.json` to make auto your personal default.

**Classifier blocks by default:**

- `curl | bash` and other code-fetch-and-exec.
- Sending sensitive data to external endpoints.
- Production deploys / migrations.
- Mass cloud-storage deletion.
- Granting IAM or repo permissions.
- Modifying shared infrastructure.
- Irreversibly destroying files that existed before the session.
- Force push; pushing directly to `main`.
- **Git rewriters that discard work:** `git reset --hard`, `git checkout -- .`, `git restore .`, `git clean -fd`, `git stash drop`, `git stash clear`.
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

**Subagents**: classifier vets the delegated task at spawn, each action during the run, and reviews the full action history on return.

**Fallbacks:**

- Each denial appears in `/permissions` → Recently denied tab; press `r` to retry with manual approval.
- After **3 denials in a row** or **20 total**, auto mode pauses and prompts. Approving resumes auto. Counters: consecutive resets on any allow; total persists until its own threshold fires.
- In `-p` (headless), repeated blocks abort the session.

Classifier runs on a server-configured model independent of your `/model`. **Counts toward token usage** and adds a round-trip on each shell/network operation (reads + working-dir edits skip it).

### `dontAsk`

Auto-deny everything that would prompt; the session never waits for input. Fully non-interactive — for CI/scripts.

**Still runs:** reads inside the working directories, the built-in read-only Bash set, calls matching `permissions.allow`, and calls a `PreToolUse` hook approved.

**Denied rather than prompted:** explicit `ask` rules, the built-in `AskUserQuestion` tool (even with a matching allow rule), connector tools your org set to `ask`, and MCP tools marked `requiresUserInteraction`.

**Can't be approved by an allow rule or a hook at all:** recognized `rm`/`rmdir` critical-path removals, and reads from a network path.

Cloud sessions on Claude Code for the web **ignore `defaultMode: "dontAsk"`**.

Set with `--permission-mode dontAsk`.

### `bypassPermissions`

Skip every check. **Includes writes to protected paths.**

Still prompts or denies — the "actions no mode auto-approves" set:
- Explicit `ask` rules, and connector tools your org set to `ask`.
- Tools requiring user interaction: `AskUserQuestion`, MCP tools marked `requiresUserInteraction`.
- Recognized `rm`/`rmdir` critical-path removals, and anything flagged `Dangerous rm operation not ruled out`. The `rm` prompt can be turned off in this mode.
- Certain cross-session messaging safeguards.
- Reads outside the working directories when `permissions.blockReadsOutsideWorkingDirectories` is on.

**Deny rules still block. Allow rules have no effect.**

**Won't enter from a session that didn't start with one of the enabling flags** — restart with `--permission-mode bypassPermissions` or `--dangerously-skip-permissions` (equivalent), or set `permissions.defaultMode` in user, `--settings` or managed settings.

- `--allow-dangerously-skip-permissions` adds the mode to the `Shift+Tab` cycle without activating it.
- `--restricted` refuses the mode outright (v2.1.248+).
- Background sessions need the acceptance recorded in user or managed settings.
- Accepting the first-run warning writes `skipDangerousModePermissionPrompt: true` to `~/.claude/settings.json`; declining exits Claude Code.

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
| Desktop | Mode selector next to send button |
| Web/mobile | Mode dropdown next to prompt; cloud sessions limited to Accept edits / Plan / Auto |

Optional modes slot into the `Shift+Tab` cycle after `plan` when enabled, **`bypassPermissions` first and `auto` last** (`bypassPermissions` requires startup with an enabling flag; `dontAsk` is never in the cycle, flag-only).

**Starting-mode precedence:** the `--permission-mode` flag, then `permissions.defaultMode` in a settings file, then the built-in default. A project's `.claude/settings.json` or `.claude/settings.local.json` is honored for terminal sessions for **every value except `auto` and `bypassPermissions`**.

## Protected paths

Writes to these are never auto-approved in any mode **except** `bypassPermissions`:

| Mode | Protected writes |
| --- | --- |
| `default`, `acceptEdits` | Prompted |
| `plan` | Allowed in an interactive terminal session where bypass is available. Otherwise routed to the classifier if auto is available during planning, else prompted |
| `auto` | Routed to classifier |
| `dontAsk` | Denied |
| `bypassPermissions` | Allowed |

**`permissions.allow` rules don't pre-approve protected-path writes** — the safety check runs before allow rules. In modes that prompt, the prompt offers "Yes, and allow Claude to edit its own settings for this session" which approves further `.claude/` writes in that session.

**Protected directories:** `.git`, `.config/git`, `.vscode`, `.idea`, `.husky`, `.cargo`, `.devcontainer`, `.yarn`, `.mvn`, `.claude`. `.claude` has exceptions: `.claude/worktrees/`, the current session's plan files, and project auto-memory Markdown files in sessions started without `--restricted`.

**Protected files:**
- Git: `.gitconfig`, `.gitmodules`.
- Shells / env: `.bashrc`, `.bash_profile`, `.bash_login`, `.bash_aliases`, `.bash_logout`, `.zshrc`, `.zprofile`, `.zshenv`, `.zlogin`, `.zlogout`, `.profile`, `.envrc`.
- Package managers: `.npmrc`, `.yarnrc`, `.yarnrc.yml`, `.pnp.cjs`, `.pnp.loader.mjs`, `.pnpmfile.cjs`, `bunfig.toml`, `.bunfig.toml`.
- Build / tooling: `.bazelrc`, `.bazelversion`, `.bazeliskrc`, `gradle-wrapper.properties`, `maven-wrapper.properties`.
- Hooks: `.pre-commit-config.yaml`, `lefthook.yml`/`.lefthook.yaml`/etc.
- Editor / lint: `.ripgreprc`, `pyrightconfig.json`, `.devcontainer.json`.
- Claude: `.mcp.json`, `.claude.json`.
- **Anything passed via `--settings`, `--mcp-config` or `--agents`** — and a settings file that is a symlink to a protected target is protected too.

## Working directories

By default Claude has access to the launch directory only. Extend with:

| Mechanism | Persistence | What it grants |
| --- | --- | --- |
| `--add-dir <path>` | One session | File access **+** loads skills, subagents, plugin keys, optionally CLAUDE.md (env-var-gated). Live reload for skills. |
| `/add-dir` | One session | Same as `--add-dir`. |
| `permissions.additionalDirectories` | Permanent (per scope) | **File access only.** No config loading from these dirs. |
| `/cd <path>` | Until next `/cd` | Relocates the session: loads new `CLAUDE.md`, `--resume` finds session from there. |

**Key distinction**: `--add-dir`/`/add-dir` *and* `additionalDirectories` both grant file access — but only the flag/command versions also load configuration. For configuration shared across projects, prefer:
- User-level (`~/.claude/agents/`, `~/.claude/settings.json`),
- Plugins, or
- Launching from the config directory directly.

## Workspace trust (project allow rules)

`permissions.allow` rules and `permissions.additionalDirectories` in a project's `.claude/settings.json` **grant** capability, so Claude Code applies them only after you accept the workspace-trust dialog for that folder. `deny` and `ask` rules aren't gated — they only restrict.

- Trust is keyed on the **git repository root**, so it covers the whole repo except nested repos (submodules). In a worktree it uses the main checkout's root.
- Started in your **home directory**, trust is held for that session only and never written to disk.
- **`claude -p` and the SDK never show the dialog.** Project allow rules and additional directories are simply not applied, and Claude Code prints a `this workspace has not been trusted` warning to stderr. Project *hooks*, `env` blocks and helper commands **do** run — see [`../tools/hooks.md`](../tools/hooks.md).
- `.claude/settings.local.json` normally skips the trust step as your own file — unless it's tracked in git or `.claude` is a symlink, in which case it's treated as repository-supplied.

`permissions.blockReadsOutsideWorkingDirectories` makes the file tools refuse paths outside your working directories in **every** mode. In auto mode Claude Code offers to turn it on the first time Claude reads outside them.

## Hooks integration

`PreToolUse` hooks register custom logic that runs before the permission prompt. Hook output can `deny`, force `ask`, or `allow`.

**Hooks don't bypass rules:**
- Deny rules and ask rules apply regardless of hook output.
- A matching deny still blocks even when the hook returned `"allow"`.
- A matching ask still prompts even when the hook returned `"allow"`.
- A hook that exits with code 2 blocks before rules are evaluated — overrides allow rules.

> **A mod can get past both.** A mod handling `tool.check` answers after `PreToolUse` hooks. With managed settings on the machine, or a Team/Enterprise `/login` as the session credential, deny rules and hook blocks hold. **Without either** — an API key or a long-lived `CLAUDE_CODE_OAUTH_TOKEN` session counts as without — a mod can approve a call a deny rule refused or a hook blocked. In plan mode it can only do that for read-only calls.

One tool is outside the system: a deny or ask rule can't block `EndConversation` while any other tool remains available.

**Use case**: bare `Bash` allow + a `PreToolUse` hook that rejects specific commands = run everything without prompts except the few you block.

## Sandbox interaction

Permissions and sandboxing are complementary:

- **Permissions** apply to all tools.
- **Sandbox** is OS-level, **Bash-only**, restricts the filesystem and network reach of subprocesses.

Combined behaviors:

- `sandbox.filesystem.allowWrite/denyWrite/denyRead/allowRead` merge with `Edit(...)` and `Read(...)` permission paths into the sandbox boundary.
- `sandbox.network.allowedDomains` / `deniedDomains` merge with `WebFetch(domain:...)` permission rules.
- With `autoAllowBashIfSandboxed: true` (default), sandboxed Bash skips the bare `Bash` ask rule prompt — the sandbox boundary substitutes. **Content-scoped ask rules like `Bash(git push *)` still prompt.** `rm`/`rmdir` targeting `/`, home, or critical system paths still prompt.
- **Plan mode skips that substitution.** Built-in read-only commands still run unprompted, but any other shell command goes through the normal permission flow while you're planning.
- Commands in `sandbox.excludedCommands` respect the bare `Bash` ask rule normally.

## Managed-only settings (org admins)

| Setting | Effect |
| --- | --- |
| `allowManagedPermissionRulesOnly` | User/project can't define allow/ask/deny. Only managed rules apply. |
| `allowManagedMcpServersOnly` | Only managed `allowedMcpServers` respected (deny still merges). |
| `allowManagedHooksOnly` | Only managed/SDK/managed-plugin hooks load. |
| `sandbox.filesystem.allowManagedReadPathsOnly` | Only managed `allowRead` paths honored. |
| `sandbox.network.allowManagedDomainsOnly` | Only managed `allowedDomains` honored; others auto-blocked. |
| `disableBypassPermissionsMode` / `disableAutoMode` | Lock off `bypassPermissions` / `auto`. `disableBypassPermissionsMode` is usually deployed as managed policy but **works from any scope** — set it in your own settings to lock yourself out. |
| `strictPluginOnlyCustomization` | Lock skills/agents/hooks/mcp to plugins + managed. |

## Precedence (recap)

Same as global settings:

1. **Managed**
2. **CLI args** (`--allowedTools`, `--disallowedTools`, `--permission-mode`)
3. **Local project** (`.claude/settings.local.json`)
4. **Shared project** (`.claude/settings.json`)
5. **User** (`~/.claude/settings.json`)

**Array merging**: `permissions.allow / ask / deny` and `additionalDirectories` merge + dedupe across scopes. If a tool is denied at any level, no other level can allow it.

## Path form in rules

`Read`/`Edit` rules use POSIX-normalized paths: `//path` is absolute, `/path` is project-relative.

`bypassPermissions` refuses to run as root/sudo unless it detects a recognized sandbox.

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
- Auto mode is worth trying eventually — but the repo-spoof rule means we cannot make this repo auto-mode-default by checking it into `.claude/settings.json`. Personal default → `~/.claude/settings.json`.
