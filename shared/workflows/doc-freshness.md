# Doc freshness: stamps and the refresh pass

Distilled docs rot. The product moves, the page you distilled from gets renamed, and the file on disk keeps asserting last quarter's facts with total confidence. Nothing in the file tells you which.

This doc defines the stamp that makes rot visible, and the pass that clears it.

## Why stamps

This playbook went four months between merges. Over that gap the model lineup advanced a generation, two source URLs 404'd, one doc ended up citing a dead page as its own provenance, and `auto` mode went from research preview to the default permission mode — a reversal that silently inverted the setup advice.

None of that announced itself. Finding it took a full manual sweep of every file. The sweep was the real cost, and it recurs every time you come back after a gap.

A stamp converts that sweep into a comparison. You stop asking "is this still true?" of 44 files and start asking "which files were verified before version X?" — a question a single command answers.

## What a stamp doesn't tell you

A stamp records **when a doc was last checked, not whether the check was right.** It catches staleness. It does nothing about a claim that was wrong the day it was written.

The worked example: `claude/tools/tools-reference.md` said the `Artifact` tool required a Team or Enterprise plan. Artifacts are available on Pro and Max too. That was wrong from the day it was written, and it survived for months because:

- It was never *drift*. No version gap, dead URL or renamed page ever pointed at it, so none of the signals in the refresh pass below could have found it.
- Step 1 narrows to docs whose stamp is behind the installed version. That doc's stamp was current, so it was never a candidate.
- It picked up a fresh stamp on **two** separate passes, because a stamp bump doesn't require re-reading every claim in the file.

It surfaced only because someone fetched the artifacts page for an unrelated reason and happened to notice.

Two consequences worth keeping in mind:

- **A stamp is per-doc, not per-claim.** One verified claim says nothing about the other forty in the same file.
- **Drift signals are necessary but not sufficient.** The pass below finds what *changed*. Nothing in it finds what was wrong to begin with.

Two cheap habits that would have caught this one:

- **Spot-check while you're there.** Any time you fetch a page for any reason, glance at the claims the repo already makes from it. Free verification, and it is how this error was actually found.
- **Suspect claims that gate access.** Plan requirements, availability matrices, permission and auth prerequisites. A wrong one fails *silently* — it removes an option instead of producing an error. A wrong command breaks loudly the first time you run it; a wrong "requires Enterprise" just quietly stops you trying.

### The 2026-10-27 cohort, and what cleared it

`verified_on: 2026-10-27` came from the bulk pass that introduced this convention, which checked specific drift patterns — model names, provider renames, dead URLs, version gates — rather than reading each doc end to end against its sources.

A verification round cleared it. Across the six reference docs plus the remainder, **128 corrections** were found against roughly 390 checkable claims — about one in three. The pattern was consistent: every doc had the right mental model and the wrong details. Not one needed its argument changed; they needed their tables corrected.

Two docs still read `2026-10-27`, and that date is **correct** rather than stale: `claude/tools/prompt-caching.md` and `claude/slash-commands/built-in.md` were genuinely read against their sources that day, and sections of both were rewritten from that reading. A stamp records when a doc was read, so bumping them to a later date would be the dishonest move.

Four docs lost their stamps entirely, under the assert-versus-delegate test above.

## The stamp

Every doc that asserts externally verifiable product facts carries YAML frontmatter:

```yaml
---
verified_against: 2.1.308
verified_on: 2026-10-27
sources:
  - https://code.claude.com/docs/en/prompt-caching
  - https://code.claude.com/docs/en/model-config
---
```

| Field | Meaning |
| --- | --- |
| `verified_against` | Claude Code version whose behaviour the claims were checked against. Bare version, no `v`. |
| `verified_on` | ISO date of that check. Catches drift in things that aren't version-gated, like a renamed cloud provider. |
| `sources` | Every page the doc was distilled from. One per line. |

`sources` is the field that earns its keep fastest. It makes provenance checkable: you can test whether the pages a doc was built from still exist without reading a word of the doc. That is how `claude/tools/context-window.md` was caught citing a 404.

### Which docs get stamped

**The test is whether the doc *asserts* product facts or *delegates* them.** A doc that says "`timeout` defaults to 600 seconds" can be wrong on its own. A doc that says "see `hooks.md` for the timeouts" can't — it rots only if `hooks.md` rots, and `hooks.md` is stamped. Stamping a delegating doc adds a verification obligation with no information behind it.

Stamp:

- Reference docs under `claude/` — the ones that state mechanics: settings keys, tool names, matcher values, flags, limits.
- `shared/project-setup/` — install steps, settings templates, command names.
- `research/anthropic/` — raw notes are the most perishable content in the repo.
- Any workflow that names real commands and flags of its own.

Leave unstamped:

- **`shared/principles/`** — durable reasoning. "Context is finite" does not expire.
- **Delegating workflows** — `plan-first.md`, `memory-curation.md`, `custom-subagent-creation.md`, `scaffolding-decision.md` and this file. They describe practice and point at the reference docs for every mechanic.
- **READMEs** — navigational. They rot when the repo's structure changes, which `git log` already tells you.
- **Root `CLAUDE.md`** — describes this repo, not the product.

Omit `sources` when a doc isn't distilled from a published page. Keep `verified_against` and `verified_on` regardless: the claims still age.

> This split was made after the first full verification round. Before it, every `claude/` and `shared/workflows/` file carried a stamp, including four that assert nothing verifiable — which meant the stale list always had four entries that no amount of checking could clear.

### Keep volatile facts out of durable docs

A principle that quotes a number inherits that number's expiry date. `shared/principles/context-is-finite.md` used to open with "Claude Code: 200K by default, 1M on some models" — two facts that drift, in a doc whose argument doesn't depend on either.

Phrase durable docs around the volatile specifics and link to the stamped doc that owns them. If you can't, the doc isn't durable and needs a stamp.

## The refresh pass

Run this when you come back to the playbook after a gap, before trusting it on a new project.

### 1. Find the stale docs

Compare every stamp against your installed version.

```bash
installed=$(claude --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')
echo "installed: $installed"
git ls-files '*.md' | while read -r f; do
  awk -v file="$f" -v inst="$installed" '
    NR==1 && $0 != "---" { exit }
    NR>1  && $0 == "---" { exit }
    /^verified_against:/ { sub(/^[^:]*:[ \t]*/, ""); v=$0 }
    /^verified_on:/      { sub(/^[^:]*:[ \t]*/, ""); d=$0 }
    END { if (v != "" && v != inst) printf "%-52s %s  (%s)\n", file, v, d }
  ' "$f"
done
```

It reads **only the leading frontmatter block** and stops at its closing `---`. That matters more than it looks: a naive `grep '^verified_against:'` also matches the example stamps inside this file and the root `README.md`, which then show up as stale docs that don't exist.

It doesn't tell you a doc is *wrong* — only that nobody has checked it since. That's the point: it narrows the sweep to a candidate list.

### 2. Check provenance before content

Dead and redirected URLs are the cheapest signal available, and a renamed page usually means the content moved too.

```bash
grep -rhoE 'https://code\.claude\.com/docs/en/[a-z0-9./-]*' --include='*.md' . \
  | sed 's/\.md$//;s/[.,)]$//' | sort -u \
  | while read -r u; do
      printf '%s %s\n' "$(curl -s -o /dev/null -w '%{http_code}' -L --max-time 15 "$u")" "$u"
    done | sort | grep -v '^200'
```

### 3. Re-distill, don't patch

For each stale doc, fetch its `sources` and re-read the claims against the live page. Patching from memory is how a doc ends up half-current, which is worse than uniformly old — you can't tell which half to trust.

Watch for the three failure modes that don't look like errors:

- **Incomplete lists.** A model or provider list that was complete when written now omits entries. It reads as correct because nothing in it is false.
- **Reversed defaults.** A feature that was opt-in becomes the default, or a preview goes GA. The mechanics survive; the recommendation inverts.
- **Renames.** "Google Vertex AI" became "Google Cloud's Agent Platform". The behaviour didn't change, so nothing breaks — the doc just stops matching what the reader sees on screen.

### 4. Prune version gates

A note like "(v2.1.142+)" earns its place only while someone might be running something older. Once the floor has moved well past it, it's noise charged to your context budget on every read.

Drop gates more than a few minor versions behind what you run. Keep them where the feature is recent enough that a reader on an older build would otherwise be confused by its absence.

### 5. Re-stamp

**A stamp means the doc was read against its sources on that date.** Bump `verified_against` and `verified_on` when that's what you did — including for docs you confirmed were already correct. An unchanged doc with a current stamp is the whole point: it's how the next pass knows to skip it.

Three cases where you should **not** bump:

- You verified one claim and changed it, but didn't re-read the rest. That's the mistake that let the `Artifact` error survive — the bump asserted a whole file had been checked when three lines had.
- You looked at the doc but couldn't reach its sources.
- You edited it without touching a factual claim: removing a section, fixing a link, rewording. Nothing about the doc's accuracy changed, so the date shouldn't move.

Leaving an old stamp is honest. It keeps the doc on the candidate list, which is where an unverified doc belongs.

### 6. Sweep the whole repo for the identifiers you touched

Once you've corrected a doc, grep the **entire repo** for every identifier you changed — setting keys, tool names, matcher values, plan names. Two reasons it keeps paying off:

- **The same claim is often in two places.** A catalog table and a bullet list 250 lines apart; a tool doc and the principle that cites it. Fixing one and stopping leaves the repo contradicting itself.
- **Sources mark things "removed", and that's the cheapest error to find.** Searching for an identifier the source says no longer exists has caught `bypass_permissions_disabled`, `teammateDefaultModel`, `TaskOutput` and `maxSkillDescriptionChars` — four errors, each a dead name sitting in a table as if it were live.

```bash
grep -rn 'identifierYouChanged' --include='*.md' .
```

### 7. Sample what nothing flagged

Steps 1 and 2 only surface drift. Before you call the pass done, verify a handful of claims that nothing pointed at — drawn from the access-gating kind above, since those are the ones that fail silently.

Sampling is a poor substitute for reading 7,000 lines, and a good substitute for reading none of them.

## Running it with an agent

The pass is well suited to delegation — it's mechanical, parallel, and read-heavy. Point an agent at this file and give it the candidate list from step 1.

Delegate the fetching and diffing per doc; a subagent reads the live page in its own context and returns only what changed. See [`research-then-edit.md`](research-then-edit.md) for why that shape matters, and [`../../claude/workflows/parallel-with-forks.md`](../../claude/workflows/parallel-with-forks.md) for fanning out across several docs at once.

Keep the final call on wording yours. An agent re-distilling from a live page will faithfully reproduce the vendor's framing, which is usually longer and less opinionated than what belongs here.

## See also

- [`../principles/durability-ladder.md`](../principles/durability-ladder.md) — where a given piece of knowledge should live in the first place.
- [`../principles/context-is-finite.md`](../principles/context-is-finite.md) — why pruning dead version gates is worth the diff.
- [`../../research/anthropic/claude-code-docs/priorities.md`](../../research/anthropic/claude-code-docs/priorities.md) — current state of the doc-surface sweep.
