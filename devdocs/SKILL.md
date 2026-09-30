---
name: devdocs
description: Captures reusable engineering knowledge from completed Shopify theme development tasks in Asana (task discussion + linked GitHub PR) and turns it into searchable, structured client documentation stored in Google Drive. Also loads that documentation back into the current Claude Code session before new related work starts, so past fixes, known Shopify limitations, workarounds, and codebase architecture don't get rediscovered from scratch. Use this skill whenever the user asks to document a completed task, log a fix, save what was learned from a PR, turn an Asana task into documentation, or wants prior knowledge loaded for a client before starting work (e.g. "get-context for Ruti", "what do we know about this client's theme", "load context", "document this task"). Shared, cross-client docs under shopify-theme/shared/ are only ever written via the explicit "/devdocs update/shopify-theme/shared" invocation, never automatically during capture. Also trigger on the bare skill name or shorthand like "/devdocs" or "/devdocs get-context/..." or "/devdocs update/...".
---

Base directory for this skill: `.claude/skills/devdocs`

# devdocs: Dev Knowledge Capture & Context Loader

Captures reusable knowledge from completed Shopify theme tasks (Asana + linked GitHub PR) into structured markdown in Google Drive, and loads it back before related work starts. **v1 scope: Shopify theme stack only** — no `shopify-apps`/`shopify-extensions` folders, and this never triggers automatically on task completion; only when a developer runs it.

## Modes

| Invocation | Does |
|---|---|
| `/devdocs [task]` | Capture — document one completed task (Asana URL/ID/name) |
| `/devdocs` | Capture — latest 5 completed tasks |
| `/devdocs get-context/shopify-theme/[client]` | Load that client's theme knowledge |
| `/devdocs update/shopify-theme/shared` | The only path that writes `shared/*.md` |

## Required connections

Needs Asana, GitHub (MCP or `gh` CLI), and Google Drive connected. Missing one → stop and say which, don't proceed from memory.

## Storage

Root folder ID: `1CfWL20rTC-EAzvI0o8hDz8EbfUsLptXj` (`Dev Engineering Docs`). Use this ID directly for every Drive read/write — never search Drive by name, and never fall back to a name search if the ID lookup fails; stop and tell the developer instead.

```
shopify-theme/
  shared/
    best-practices.md   # canonical "the best way to build X" — broad, prescriptive
    patterns.md          # narrower how-to recipes, workarounds, hard platform limits
  themes/
    [client]/changelog.md   # the only file a client ever has
```

`shared/*.md` describes techniques generically — no client file names or paths, ever. `changelog.md` is the opposite — it's one specific codebase, so real file paths belong there. Slack is never used in this skill.

## Mode 1: Capture

1. **Resolve the task(s).** Given one, resolve via Asana (URL/ID/name search). Given none, use the latest 5 completed tasks. Skip anything not Complete/Done and say why. Keep each Asana task ID — it's the dedup anchor.
2. **Identify the client.** Fetch the full task (description, comments, subtasks, custom fields, linked PR from any custom field/attachment/comment). Parse `[Client]` from the title, normalize to lowercase-hyphenated (`Ruti`→`ruti`). If the folder doesn't exist under `themes/`, create it — flag a possible typo without blocking.
3. **Gather context**, in order: Asana (intent) → GitHub PR (the real diff/commits/review comments — usually where the actual fix lives) → this client's `changelog.md` (dupes/related entries) → `shared/*.md` (known limitation/pattern overlap). No PR linked → document from Asana content alone, and flag the missing PR in the confirmation summary (step 6) rather than blocking.
4. **Classify** — exactly one:

   | Category | When |
   |---|---|
   | Bug Fix | Something was broken |
   | New Pattern | Reusable narrow technique used/built; promotable to `patterns.md` |
   | Shopify Limitation | Platform constraint, with or without workaround; promotable to `patterns.md` |
   | Best Practice | Canonical way to build a whole category of thing; promotable to `best-practices.md` |
   | Discovered Understanding | No new code, just real understanding gained |
   | Skip / Low Value | Trivial, no reusable knowledge |

   Default to documenting when unsure. Add 2-5 searchable tags.
5. **Write `changelog.md`.** Always touched, every task, every category — Skip/Low Value still gets logged, just without a rich entry. Search for `<!-- asana-task-id: [ID] -->`; found → update in place, not found → append using this template:

   ```markdown
   ## [Task Name] — [Asana Link]

   <!-- asana-task-id: 123456789012345 -->

   **Date:** YYYY-MM-DD
   **Category:** Bug Fix / New Pattern / Shopify Limitation / Best Practice / Discovered Understanding / Skip (Low Value)

   **Problem:** what was broken or needed
   **Why:** root cause or reasoning
   **What we did:** the actual change, with file/PR references — not full diffs

   **PR:** link
   **Files touched:** list
   **Tags:** tag1, tag2, tag3
   **See also:** related entries, this client or others

   **Promoted to:** shared / —

   **Notes / gotchas:** anything a future dev should know
   ```

   Never touch `shared/*.md` here — even a clearly generalizable task just gets a note under **Notes / gotchas** ("candidate for shared/patterns.md") and a mention in the summary. Reference file/PR links, never paste full code.
6. **Confirm.** One summary: tasks processed, skipped (+why), category per task, docs created/updated (links), typo flags, missing-PR flags.

## Mode 2: `get-context/shopify-theme/[client]`

Loads `shared/best-practices.md`, `shared/patterns.md`, and `themes/[client]/changelog.md` — skip silently whichever don't exist yet. Unknown client → say so and list the clients that do exist, don't guess.

This isn't a one-time read: actively hold what's loaded against whatever the developer asks next, and flag a match (a tag, a limitation, a workaround) before being asked, even if they didn't reference it.

## Mode 3: `update/shopify-theme/shared`

The only writer of `shared/*.md`. Uses the task/PR context already established this session — never re-fetches from Asana. No context yet → stop, ask for a task first.

- Pick the file: `best-practices.md` for a full canonical guide, `patterns.md` for a narrower recipe or platform constraint.
- Update in place (merge with a related entry, don't duplicate), written generically — no client names/paths.
- Then set that task's changelog entry to `Promoted to: shared`, and confirm which file changed and what was added.

## A few things not to do

- Don't write to `shared/*.md` during Capture, ever — only Mode 3 does that.
- Don't create `shopify-apps`/`shopify-extensions` folders or other non-theme stacks.
- Don't fall back to a Drive name search if the pinned root folder ID lookup fails — stop and tell the developer.
- Don't duplicate a changelog entry — always check the `asana-task-id` anchor first.
- Don't paste full diffs/code into `changelog.md` — reference file paths and PR/commit links instead.
