---
name: devdocs
description: Captures reusable engineering knowledge from completed Shopify theme development tasks in Asana (task discussion + linked GitHub PR) and turns it into searchable, structured client documentation stored in Google Drive. Also loads that documentation back into the current Claude Code session before new related work starts, so past fixes, known Shopify limitations, workarounds, and codebase architecture don't get rediscovered from scratch. Use this skill whenever the user asks to document a completed task, log a fix, save what was learned from a PR, turn an Asana task into documentation, or wants prior knowledge loaded for a client before starting work (e.g. "get-context for Ruti", "what do we know about this client's theme", "load context", "document this task"). Shared, cross-client docs under shopify-theme/shared/ are only ever written via the explicit "/devdocs update/shopify-theme/shared" invocation, never automatically during capture. Also trigger on the bare skill name or shorthand like "/devdocs" or "/devdocs get-context/..." or "/devdocs update/...".
---

# devdocs — Dev Knowledge Capture & Context Loader

## Purpose

Completed Shopify theme work carries knowledge that normally lives only in a PR, a Slack message, or a developer's head — the actual root cause of a bug, a Shopify platform limitation that was worked around, or a piece of theme architecture someone had to reverse-engineer. This skill turns that knowledge into structured, searchable markdown files in Google Drive, and loads it back into a session before a developer starts related work.

**v1 scope — this skill only covers the Shopify theme stack.** Do not attempt to create `shopify-apps`, `shopify-extensions`, or other stack folders — those are deliberately out of scope until this is extended later. Do not attempt to trigger automatically when an Asana task is marked complete — this skill is invoked manually, only when a developer runs it.

## Three invocation modes

| Invocation | Mode |
|---|---|
| `/devdocs [task]` | **Capture** — document a specific completed task |
| `/devdocs` (no argument) | **Capture** — document the latest 5 completed tasks |
| `/devdocs get-context/shopify-theme/[client]` | **Load** — theme knowledge for one client |
| `/devdocs update/shopify-theme/shared` | **Update Shared** — the only path that ever writes to `shared/*.md` |

If the argument doesn't match `get-context/...` or `update/...` and isn't empty, treat it as a task reference (Asana URL, Asana task ID, or a task name to search for).

**`shared/*.md` is never written during Capture.** Capture only ever touches a client's `changelog.md` (always) and `architecture.md` (when applicable). Promoting something to the shared docs happens only through the explicit `/devdocs update/shopify-theme/shared` invocation — see Mode 3 below.

## Required connections

This skill needs three MCP connections available in the current session: an **Asana** connector, a **GitHub** connector (or the `gh` CLI via the terminal if no GitHub MCP is connected), and a **Google Drive** connector.

Before doing anything else, confirm these are available. **If any required connection is missing, stop immediately and tell the developer which one is missing** — do not proceed with partial context, and do not guess at task or PR details from memory.

## Configuration

**Dev Engineering Docs root folder (Google Drive):** `https://drive.google.com/drive/u/0/folders/1CfWL20rTC-EAzvI0o8hDz8EbfUsLptXj` (folder ID: `1CfWL20rTC-EAzvI0o8hDz8EbfUsLptXj`)

Use this folder ID directly as the root for every Drive read/write in this skill — do not search Drive by the name "Dev Engineering Docs" first. Under this root, the `shopify-theme/` folder and `shopify-theme/shared/` already exist; client folders under `shopify-theme/themes/` get created as needed, per Stage 2 below.

If a lookup against this folder ID ever fails (folder moved, permissions changed, etc.), stop and tell the developer rather than falling back to a name search that could land in the wrong "Dev Engineering Docs" if more than one exists.

### Shared docs are theme-agnostic — client docs are not

`shopify-theme/shared/*.md` describes techniques that should hold up on *any* custom-built theme, not just the one a task happened to come from. Two different clients' themes can implement the same idea with completely different file names and structure. Because of this:

- **Never reference a specific file name or path in a `shared/` doc.** Describe the technique itself, generalized, with an illustrative code snippet if useful. If something is genuinely tied to one client's specific implementation (not a portable technique), it belongs in that client's `architecture.md`, not in `shared/`.
- **Client docs (`architecture.md`, `changelog.md`) are the opposite** — reference real file paths there. Those docs describe one specific codebase, so pointing at the actual file is exactly what makes them useful.
- `shared/` has **two** files, each with a distinct scope — getting a task into the right one matters more than getting it into *a* shared file:
  - **`best-practices.md`** — the canonical, definitive way to build a whole category of thing. If someone asks "what's the best way to build X," the answer lives here. Broad and prescriptive, e.g. "the best way to build a new section" (a full boilerplate/skeleton). Not for narrow tips — those go below.
  - **`patterns.md`** — reusable how-to recipes for a specific, narrower need. Covers plain "how to do X" recipes, constraint-with-a-fix entries (workarounds), and hard platform constraints with *no* workaround (the limitation is just a fact to design around) — all folded into this one file rather than split further. This is the shared home for **New Pattern** and **Shopify Limitation** from Stage 4, once either generalizes beyond one client.

Slack is **not** used anywhere in this skill, even if a Slack connector is available in the session.

---

## Mode 1: Capture & Document

Work through these stages in order for every task being processed.

### Stage 1 — Resolve which task(s) to process

- **Task given:** resolve it via the Asana connector (accept a task URL, a raw task ID, or a name to search for).
- **No task given:** fetch the **latest 5 completed Asana tasks**, sorted by completion date, most recent first.
- Whichever path, confirm the task's status is Complete/Done. If a specifically-named task isn't complete, tell the developer and stop for that task rather than documenting in-progress work.
- Keep the Asana task ID for every resolved task — it's the deduplication anchor used later.

### Stage 2 — Identify the client

- Fetch the full task: title, description, all comments, all subtasks and their comments, custom fields, assignee, and any linked PR (check a custom field, an attachment, and the comments — PR links show up in any of these).
- Parse the `[Client]` tag from the task title, e.g. `[Ruti] Fix sticky header` → client is `Ruti`. Normalize it to a filesystem-safe folder name (lowercase, hyphenated: `rival-activewear`, `still-here`, `ruti`).
- If the normalized name doesn't match an existing client folder under `shopify-theme/themes/`, create the folder — but also flag it to the developer as a possible typo (e.g. `[Stil Here]` → "did you mean `[Still Here]`?"). **Do not block on this** — create the folder and continue, just surface the warning in the final summary.

### Stage 3 — Gather context

Pull from exactly these sources, in this priority order:

1. **Asana** — task intent (already fetched in Stage 2)
2. **GitHub PR** — the actual implementation: diff, commit messages, review comments, files changed. This is usually where "how it was really fixed" lives, not in the Asana description.
3. **This client's existing docs** (`architecture.md` + `changelog.md`) — check for related or duplicate prior entries.
4. **Shared theme docs** (`shopify-theme/shared/*.md`) — check whether this overlaps a known platform-wide limitation, workaround, or best practice.

Merge everything into one working understanding: problem, root cause / reasoning, what was actually done, key files, and anything a future dev would want to know before touching this area again.

### Stage 4 — Classify

Assign exactly one category:

| Category | When to use it |
|---|---|
| **Bug Fix** | Something was broken; capture problem, root cause, fix |
| **New Pattern** | A reusable technique for a specific, narrower need was used or built (not a full "how to build X" guide — that's Best Practice). If portable beyond this client, promotes to `shared/patterns.md`. |
| **Shopify Limitation** | A platform constraint was discovered, with or without a workaround. If portable, promotes to `shared/patterns.md`. |
| **Best Practice** | The canonical, definitive way to build a whole category of thing — broad and prescriptive, not a narrow tip. If portable, promotes to `shared/best-practices.md`. |
| **Discovered Architecture** | No new code, but real understanding of how an existing area works was gained (e.g. had to reverse-engineer how the cart drawer talks to an app) |
| **Skip / Low Value** | Trivial — copy change, typo fix, no reusable technical knowledge |

Default toward documenting when it's unclear whether something is useful — losing knowledge silently is worse than one extra changelog entry. Assign 2-5 short searchable tags (e.g. `scroll`, `pdp`, `sticky-header`, `liquid`).

### Stage 5 — Write or update documentation

**`changelog.md` is always touched**, for every task in every category (including Skip/Low Value — it stays auditable, it just doesn't get a rich entry). Deduplicate using the Asana task ID:

- Search `changelog.md` for an existing `<!-- asana-task-id: [ID] -->` anchor matching this task.
- **Found →** update that entry in place. Do not create a duplicate.
- **Not found →** append a new entry at the end of the file, using the exact template below.

```markdown
## [Task Name] — [Asana Link]

<!-- asana-task-id: 123456789012345 -->

**Date:** YYYY-MM-DD
**Category:** Bug Fix / New Pattern / Shopify Limitation / Best Practice / Discovered Architecture / Skip (Low Value)

**Problem:** what was broken or needed
**Why:** the root cause, or the reasoning behind the decision
**What we did:** the actual change, with code/file references — not full diffs

**PR:** link
**Files touched:** list
**Tags:** tag1, tag2, tag3
**See also:** links to related entries, this client or others

**Promoted to:** architecture.md / shared / — (leave as "—" if neither)

**Notes / gotchas:** anything a future dev should know before touching this area again
```

After writing the changelog entry, ask one more question about the same task:

- **Does this task touch an area worth reflecting in `architecture.md`?** If yes, update (don't append — edit in place) the relevant section, organized by area (Header, PDP, Cart, Checkout, etc.). Ground this update by **scanning the actual current files in that area of the client's theme** (the files the PR touched, plus their immediate dependencies — the section/snippets/assets that make up that area today), not just by restating the PR diff. The task is the trigger, but the doc should describe how the area actually works right now, so it doesn't silently drift from the live codebase over time. Set `Promoted to: architecture.md` in the changelog entry.

Capture never touches `shopify-theme/shared/*.md` — even if this task looks generalizable beyond one client, leave shared docs alone here. Note that in the changelog entry instead (e.g. under **Notes / gotchas**, something like "may generalize — candidate for shared/patterns.md") so it isn't lost, and tell the developer in the final summary that a task looks like a shared-doc candidate. Promoting it is a separate, explicit step — see Mode 3.

Reference file paths and PR/commit links instead of pasting full code — pointers stay accurate as code changes; pasted snippets go stale.

### Stage 6 — Store and confirm

- Everything lives under the root folder ID pinned in **Configuration** above. Create the client folder and its two files if they don't exist yet: `shopify-theme/themes/[client]/architecture.md` and `.../changelog.md`. Capture never creates or touches `shared/` files — see Mode 3.
- After processing every resolved task, give the developer one summary covering: how many tasks were processed, how many were skipped (and why), the category assigned to each, links to the docs that were created or updated, any client-tag typo warnings from Stage 2, and any tasks that were missing a PR link or other expected context.

---

## Mode 2: Load Context (`get-context`)

### Parsing the argument

- `get-context/shopify-theme/[client]` — load exactly this client's theme knowledge (the only supported form; v1 covers only the `shopify-theme` stack, so there's no other stack to merge in).

### What gets loaded, for a theme-stack client

```
shopify-theme/shared/best-practices.md
shopify-theme/shared/patterns.md
shopify-theme/themes/[client]/architecture.md
shopify-theme/themes/[client]/changelog.md
```

Four files. If any don't exist yet (a brand-new client, or shared files not yet started), skip them silently rather than erroring — an empty result for one file isn't a failure.

### Making the load actually useful

Loading these files into context is not the goal by itself — the goal is that they change how you respond to what the developer does next. Concretely:

- After loading, **actively hold this knowledge against whatever the developer asks next**, even if they don't reference it. If they describe a task or start writing code that touches an area covered in `architecture.md`, or matches a tag/pattern in `changelog.md`, a limitation, or a workaround, **say so before proceeding** — don't wait to be asked "does this relate to anything we've seen before?"
- If the client is unknown (no folder exists yet), say so plainly rather than silently returning nothing — the developer needs to know there's no prior knowledge to draw on for this client.
- If an unrecognized client name is given, list the clients that do exist under `shopify-theme/themes/` instead of guessing which one was meant.

---

## Mode 3: Update Shared Docs (`update/shopify-theme/shared`)

This is the **only** invocation that writes to `shopify-theme/shared/*.md`. Capture (Mode 1) never touches these files, no matter how generalizable a task looks — promoting knowledge to shared is always this separate, explicit step.

- Run this using **the task context and PR already established earlier in the current session** (from a Capture just run, or from whatever task the developer has been discussing/working on) — do not go re-fetch a task from Asana for this invocation. If no task/PR context exists yet in the session, stop and tell the developer to give a task first (run Capture on it, or describe it) before calling this.
- Decide which shared file the knowledge belongs in, same rules as before:
  - **`best-practices.md`** — a full, canonical "best way to build X" guide.
  - **`patterns.md`** — a reusable how-to recipe, or a platform constraint (with or without a workaround).
- Update the relevant file in place (don't just append) — merge with any existing related entry rather than creating a near-duplicate.
- **Write the entry generically — no client file names or paths** (see Configuration's "Shared docs are theme-agnostic" note). Describe the technique or constraint itself, with an illustrative snippet if useful.
- After updating, go back to that task's `changelog.md` entry (found via its `<!-- asana-task-id: ... -->` anchor) and set `Promoted to:` to include `shared` (alongside `architecture.md` if that was also set).
- Confirm to the developer which shared file was updated and what was added.

---

## Folder structure

```
Dev Engineering Docs/
└── shopify-theme/
    ├── shared/
    │   ├── best-practices.md
    │   └── patterns.md
    └── themes/
        ├── ruti/
        │   ├── architecture.md
        │   └── changelog.md
        ├── rival-activewear/
        │   ├── architecture.md
        │   └── changelog.md
        └── still-here/
            ├── architecture.md
            └── changelog.md
```

This structure lives under the root folder ID pinned in **Configuration** above — never search Drive by name for "Dev Engineering Docs" or create a new top-level folder for it.

---

## Quick reference: edge cases

| Situation | What to do |
|---|---|
| Required MCP connection missing | Stop immediately, tell the developer which one |
| Task not complete | Skip it, report why, don't document in-progress work |
| No PR linked to the task | Still document from Asana content alone; flag the missing PR in the confirmation summary |
| Client tag looks like a typo | Create the folder anyway, flag the possible typo, don't block |
| Task already has a changelog entry (same Asana ID) | Update in place, never duplicate |
| `get-context` for a client with no docs yet | Say so plainly — don't return an empty result silently |
| Task touches something that isn't really dev work | Classify as Skip / Low Value, still log it, don't create a rich entry |
| Task looks generalizable during Capture | Note it as a shared-doc candidate in the changelog/summary; do NOT write to `shared/` — that only happens via `update/shopify-theme/shared` |
| `update/shopify-theme/shared` called with no task/PR context in session | Stop and ask the developer to establish a task first |
