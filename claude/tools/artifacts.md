---
verified_against: 2.1.308
verified_on: 2026-10-30
sources:
  - https://code.claude.com/docs/en/artifacts
---

# Artifacts — publishing a page from a session

An artifact is a live page Claude Code publishes from your session to a private URL on claude.ai. It's private until you share it, it updates in place as the session continues, and every publish is a version you can point viewers at.

**Whether to publish at all is [`form-follows-use.md`](../../shared/principles/form-follows-use.md).** This doc is the mechanics.

Proportionality first: this matters a lot for work with a visual surface — mockups, dashboards, layout comparisons, annotated diffs — and barely at all for a backend service with no visual output. If your project is the latter, read the availability section and move on.

## What it costs

A styled page uses more output tokens than the same content as terminal text. The main contributors, in order: images embedded as data URIs, inline JavaScript for interactive controls, inline CSS.

To keep it down: prefer SVG or plain HTML/CSS for diagrams over raster images, skip interactivity you won't use, and have the page summarise a large dataset rather than inlining it. This is the [`context-is-finite`](../../shared/principles/context-is-finite.md) tradeoff in a different costume — the page is cheap to *read* and expensive to *produce*.

## What a page can and can't do

One self-contained static page, served under a strict CSP. These decide feasibility before you ask:

| Constraint | Consequence |
| --- | --- |
| **No backend** | It can't authenticate viewers or hold server state. |
| **Single page** | No routes. Relative links don't resolve, so multi-section content uses in-page anchors. |
| **External loads** | Google Fonts, plus scripts from five CDN hosts (cdnjs, unpkg, jsDelivr `/npm/`, Tailwind, jQuery). Every other external script, stylesheet, font and image is blocked; other CSS and JS is inlined and images embedded. |
| **Size** | 16 MiB rendered. Large embedded images are the usual cause of a failure. |
| **Source** | `.html`, `.htm` or `.md`, decoding as UTF-8. Markdown renders as a styled document page. |

For a hosted internal tool with a backend, deploy it yourself — an artifact is a capture of work, not a deployment.

## Availability — and the silent failure

Needs **Pro, Max, Team or Enterprise**, and a session signed in with `/login`.

It won't work, and the failure mode is Claude writing a local HTML file or saying it can't publish, when:

- The session authenticates with an **API key, gateway token or cloud-provider credential**.
- The provider is **Bedrock, Google Cloud's Agent Platform or Microsoft Foundry**.
- The organisation has CMEK, HIPAA or Zero Data Retention enabled.
- You're in the **Agent SDK, the GitHub Action, or `-p` from your own scripts** — off by default in all three.

That last one matters: **anything you build around artifacts won't run in CI.** Don't design a pipeline step on them.

## Claude publishes on its own

Claude may publish without being asked when the output suits a page, and a new artifact from a prompt you typed goes through with **no permission prompt and no classifier review**. Pages are private by default, so this isn't dangerous — but it is the agent creating external, shareable state unprompted, which is worth deciding about rather than discovering.

It does ask first in plan mode, when the page declares runtime capabilities (connector calls, file downloads), when the artifact is already shared publicly, and when it can't read the source file without your approval.

Turning it off, if you'd rather it never happened:

| Where | What |
| --- | --- |
| `/config` | Artifacts row off — writes `"enableArtifact": false` to user settings |
| Settings file | `"enableArtifact": false` |
| Environment | `CLAUDE_CODE_DISABLE_ARTIFACT=1` |
| Permissions | `Artifact` in `permissions.deny` |

A project `.claude/settings.json` can turn artifacts **off** but not back on.

## Comment watching

After your session publishes, it **watches that artifact for comments**. When someone who can edit it sends a comment to Claude, it arrives in your session, and depending on your permission mode Claude may read the thread and reply on its own.

`/tasks` lists the watches (under **System tasks** for the ones Claude Code started itself). Ctrl+C at an idle prompt pauses replying until your next message; stopping the task in `/tasks` stops it for that artifact until you ask it to resume.

## Sharing

Private to you by default. Share from the page header.

- **Within your organisation** (Team/Enterprise): specific people, or everyone. Viewers by default; editors on Team/Enterprise can republish — they give Claude the URL in their own session.
- **Publicly**: a link anyone can open with no sign-in. Off by default on Team/Enterprise until an Owner enables it.

An editor publishing while your Claude holds an older copy causes a refused publish; Claude then merges onto the newer version and publishes again.

## Updating, and finding one again

Ask for a revision and Claude edits the file and republishes to the same URL. Anyone with the page open sees it change.

From a **different session**, give Claude the URL or attach it with `/artifacts` — without either, you get a new artifact instead of an update. `/artifacts` lists what you own and what's shared with you, and works after `/clear` because it reads your claude.ai account.

`Ctrl+]` reopens the session's most recent artifact. `CLAUDE_CODE_ARTIFACT_AUTO_OPEN=0` stops the browser opening on publish.

## Design comes from your project

Claude applies a design skill when building a page, and it looks for a design system in your project before choosing its own. Precedence: your prompt, then your design system, then its defaults.

Record colours, typography and spacing in `CLAUDE.md` once and every page matches your product — see the design-tokens step in [`new-project.md`](../../shared/project-setup/new-project.md).

## Documents go elsewhere

Claude Docs reaches Claude Code as a claude.ai **connector**, not a command — `/mcp` lists it as `claude.ai Claude Docs`. With it connected, a request for a document meant for other people (a spec, a proposal, a write-up of the plan) goes there instead of becoming a page.

A document that belongs in the codebase stays a file. To force a file, name the format — `.docx`, or a Markdown file in the repo.

## Templates

`/slides` and `/design` start from claude.ai templates rather than building a page from scratch: a deck, or a canvas of artboards you can edit in the browser and export as PNG/PDF. Both are beta, on by default on Pro/Max/Team, per-template opt-in on Enterprise.

## See also

- [`../../shared/principles/form-follows-use.md`](../../shared/principles/form-follows-use.md) — whether to publish at all.
- [`mcp.md`](mcp.md) — connector-backed pages re-fetch data on each view, through the *viewer's* account.
- [`../surface-map.md`](../surface-map.md) — the rest of the surface.
