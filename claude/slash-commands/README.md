# slash-commands/

Everything typed with `/` at the start of a message. Three sources: built-in commands, bundled skills (ship with Claude Code), and custom skills (you write them).

## Current docs

- **[`built-in.md`](built-in.md)** — reference for the ~95 built-in commands and bundled skills, grouped by workflow phase (first session / during task / parallel / before ship / between sessions / when wrong).

## Where custom commands live

**Custom slash commands are skills.** A file at `.claude/commands/deploy.md` and a skill at `.claude/skills/deploy/SKILL.md` both create `/deploy` and work the same way. Skills add: a directory for supporting files, frontmatter for invocation control, and automatic discovery by Claude.

Full skill mechanics — `SKILL.md` format, frontmatter reference, lifecycle, `disable-model-invocation`, dynamic `!\`command\`` injection, evals — are covered in **[`../tools/skills.md`](../tools/skills.md)**.

## Cross-OS reminder

Most commands are OS-neutral; a few (`/desktop`, `/sandbox`, `/heapdump`, `/setup-bedrock`, `/setup-vertex`) are platform- or environment-gated. Flagged inline in `built-in.md`.
