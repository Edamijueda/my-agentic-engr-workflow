# Form follows use

The shape of an output should match how it will be used. Agents can produce terminal text, files, published pages, collaborative documents and decks — and picking the wrong one is one of the easier ways to waste the work entirely.

## The four destinations

| The output is… | It belongs in |
| --- | --- |
| **Answering the question you just asked** | the terminal |
| **Part of the codebase** — a README, a spec you'll version and diff, config | a **file** |
| **Something to look at or interact with** — a diff walkthrough, a dashboard, several options side by side | a **published page** |
| **Something other people will edit as text** — a proposal, a write-up of the plan you just agreed | a **collaborative document** |

Most output belongs in the terminal. The other three are for work that outlives the session, and the question that sorts them is simply *who touches this next, and how?*

## Both failure directions cost you

**Under-reaching.** Dumping a four-way comparison into the terminal as prose, when a page showing the options side by side would have settled it in seconds. Anything where the shape of the information *is* the information — a diff with annotations, a layout, a trend, a set of tradeoffs — loses most of its value as a wall of text. You end up re-reading it, or asking for a summary of the summary.

**Over-reaching.** Publishing a page nobody needed. A styled page costs more to produce than the same content as text, and it creates something that now exists, has a URL, and may need updating. If the answer is three sentences, three sentences is the right form.

The asymmetry worth knowing: under-reaching wastes *your* time re-reading, over-reaching wastes tokens and leaves clutter behind. Neither is free, but the first is the more common mistake.

## How to apply

- **Ask who reads it next.** You, right now → terminal. A teammate who needs to see it → page. A teammate who needs to change the words → document. The compiler or the next developer in the repo → file.
- **Say the destination when you assign the work.** An agent guessing the medium is guessing, and a good agent will sometimes guess "page" for something you wanted as two lines.
- **A document that belongs in the repo stays a file.** A README is not a collaborative document just because other people read it. If it should be versioned and reviewed in a diff, it's a file.
- **Don't let the medium drive the content.** "Make a dashboard" invites an agent to fill a dashboard. Ask for the finding first; choose the form once you know how big the finding is.
- **A rendered page can be a check.** Something you can look at next to the mockup is evidence in the sense [`checks-over-claims.md`](checks-over-claims.md) means it.

## Why this is agent-agnostic

Every one of these destinations exists outside any single tool, and the judgement doesn't change when the tool does. What changes is the mechanism: which command publishes a page, what that page can and can't do, what it costs. That part belongs in the per-agent docs and gets linked rather than restated here.

## See also

- [`../../claude/tools/artifacts.md`](../../claude/tools/artifacts.md) — publishing a page with Claude Code: constraints, cost, availability, and the collaborative-document route.
- [`context-is-finite.md`](context-is-finite.md) — why a styled page isn't free.
- [`checks-over-claims.md`](checks-over-claims.md) — output you can look at as a form of evidence.
