---
name: lets-work
description: Single session-long orchestrator for this workspace - loads your open Asana work (or a named task/project), resolves the client project (store handle + local folder) and starts/reuses the dev server, then - when run again once work looks done - drafts a commit message, pushes, opens a PR, and files DQA/FQA QA subtasks with the preview link. Replaces start-day, task, and shopify-project-resolution as separate skills; this is the only entry point for "what am I working on", "start my day", loading a specific Asana task's context, or resolving which project/store/folder is active. Session-scoped - it tracks the active task and resolved project from earlier in this same conversation, it does not persist across separate Claude Code sessions. Use when the user says "let's work", "start my day", references a task name/keyword/Asana URL, or re-invokes /lets-work partway through a task to move it to QA.
disable-model-invocation: true
---

# /lets-work - Session Workflow Orchestrator

The single skill for this workspace covering project/store resolution, session start-of-day setup, Asana task-context loading, and the dev-to-QA handoff. Net-new relative to `add-to-qa-retainer` (doesn't reuse its code) but inherits several hard-won constraints from that skill's learnings log - see "Constraints carried over" below.

## Determine the phase before doing anything

Each invocation must first work out where the active task/project stands - never assume this is always a "start" or always a "finish" call.

- **No active project/task tracked yet in this conversation** -> Phase A (Start).
- **An active task is already tracked in this conversation and the user names something different this time** (a different task/project/keyword) -> treat it as a fresh Phase A run for that new thing. Don't silently drop the old one - mention it's still sitting mid-flight before moving on.
- **An active task is already tracked, the dev server was already started, no PR opened yet, and the user re-invokes with no new task named** -> **ask first: "Are we still working on [task name]?"**
  - **Yes** -> pull back in whatever's already known from earlier in this conversation (task, project, branch) and go straight to Phase B - don't re-fetch or re-resolve what's already known.
  - **No** -> drop it as the active task and start fresh from Phase A step 1, as if nothing were tracked.
- **A PR is already open and QA subtasks already exist for the active task** -> nothing left to automate; tell the user so rather than re-running Phase B's side effects a second time.
- **Fresh Phase A run, dev-server command just handed over** -> Phase A step 6 itself asks whether development is complete and changes are staged; a "Yes" there flows straight into Phase B within the same invocation, no re-invocation of `/lets-work` needed. A "Not yet" stops the skill there, same as before - the re-invocation branches above are for picking the thread back up on a later `/lets-work` call.

Never repeat a step already completed and confirmed earlier in this same conversation (re-fetching the same Asana task, re-resolving an already-known project, re-starting an already-running dev server, etc.) - check what's already known first.

**Whenever this skill needs a yes/no (or any small fixed-choice) answer from the user** - committing, resuming the active task, Project vs Retainer, dev-server-up, etc. - ask it via `AskUserQuestion` with clickable options, never a plain text question. It's a faster, lower-friction confirmation for the user than typing a reply, and keeps this behavior consistent across every checkpoint in the skill rather than just the one it was first noticed on.

## Shopify project mapping

Use this table to resolve a project name, store handle, or client name to the other two. Not every mapped project has a local folder in this workspace yet - check `client-theme/` before assuming one exists.

| Project | Store Handle | Local Folder |
| --------------------------- | ------------------------ | ----------------- |
| GIR | guestinresidence | GIR |
| CR | crn-dev | CR |
| Capegrey | capegrey | Capegrey |
| Splits59 | splits-59 | Splits59 |
| ReDone | redun-com | ReDone |
| Still Here | still-here-new-york | Still Here |
| Rival Activewear | rivalactivewear | Rival (`client-theme/shopify-rival-activewear`) |
| Ruti | ruti-staging | Ruti (`client-theme/shopify-ruti`) |
| Shopko Optical | mmr7xd-8z | Shopko |
| Fielmann Sandbox | fielmann-sandbox | Fielmann |
| Oribe US | oribe-usa | Oribe |
| Oribe CA | oribe-canada | Oribe |
| Oribe US/CA Sandbox | oribesandbox-us | Oribe |
| Oribe UK | oribe-uk | Oribe |
| Oribe SE | oribe-sweden | Oribe |
| Oribe DE | oribe-germany | Oribe |
| Oribe EMEA Sandbox | oribehaircare-germany | Oribe |
| Kao Family of Brands USA | kaomallshop | Kao |
| Kao Beauty Brands Sandbox | kaosandbox | Kao |
| Kao Family of Brands Canada | kao-ccb-ca | Kao |
| MB South Africa | moltonbrown-south-africa | Molton Brown |
| MB Malaysia | moltonbrown-malaysia | Molton Brown |
| MB Netherlands | moltonbrown-netherlands | Molton Brown |
| MB Hungary | moltonbrown-hungary | Molton Brown |
| MB Italy | moltonbrown-italy | Molton Brown |
| MB Cyprus | moltonbrown-cyprus | Molton Brown |
| MB Poland | moltonbrown-poland | Molton Brown |
| MB Spain | moltonbrown-spain | Molton Brown |
| MB Sweden | moltonbrown-sweden | Molton Brown |
| MB Greece | moltonbrown-greece | Molton Brown |
| MB Sandbox | moltonbrown-sandbox | Molton Brown |
| Quickstart | quickstart-cf48f706 | Quickstart |

Only `blackandblackcreative`, `shopify-rival-activewear`, and `shopify-ruti` currently have a local folder under `client-theme/`. Never invent a project/store/folder that isn't in this table or in `client-theme/` - ask if something doesn't match.

## Phase A - Start

1. Figure out what the user gave you when invoking, and fire the matching Asana fetch(es) **in the same turn** as confirming connectivity (`get_me`) - don't gate the fetch on `get_me` succeeding first; connectivity failure is rare and the calls don't depend on each other:
   - **Nothing, or anything short of an exact task name/keyword/URL/GID** -> call `get_me` and `get_my_tasks(completed_since="now")` together in one turn - don't ask the user to supply a task name, keyword, or Asana link first. Sort Overdue > Today > Upcoming > No due date, exclude anything clearly completed/cancelled/blocked, and ask the user to pick one from that list. This is the default path and should fire with zero round-trips beyond the pick.
   - **A task name, keyword, Asana URL, or GID was actually given** -> call `get_me` and `search_tasks`/`get_task` together in one turn; if multiple plausible matches come back, show a short list and ask which one.
   - **A project/store/client name with no specific task** -> just confirm `get_me` and skip straight to project resolution below; there may not be an associated Asana task at all (e.g. general theme work).
2. If a task was found, load its full context (`get_task` with description, comments, attachments, custom fields, parent, project/section, due date, assignee). Note any Figma links or acceptance criteria. **Retain its gid as "the active task" for the rest of this conversation** - every later phase refers back to this same task, not a re-resolved one.
   - If the project/store is already resolvable at this point without waiting on this fetch - the user explicitly named the project/store, or the task-search result from step 1 already surfaced the project - **kick off locating the theme directory (step 4 below) in parallel with this `get_task` call** instead of waiting for it to finish first. Otherwise (project only resolvable from a field on the full task record), sequence as before: finish this fetch, then resolve the project, then locate the theme directory.
3. Resolve the client project (store handle + local folder) from: what the task/user explicitly named, the project's own field on the Asana task, or the mapping table above. Don't guess - if resolution is ambiguous (task spans projects, name doesn't match the table, no local folder exists yet), ask.
3a. If a task was selected (not just a bare project), move it into active development: set its Status custom field to **In Dev**. Check live via `get_project` (on the task's own project) for the field's gid and the "In Dev" enum option's gid before setting it - don't assume it matches another client's setup. If the task's project has no Status field, or no option that reads "In Dev", say so and skip rather than inventing one. **This lookup is independent of locating the theme directory (step 4) - run both in the same turn rather than sequentially.**
4. Locate the theme directory by checking where the session is **already sitting** - `client-theme/<Local Folder>` is just this one workspace's own layout, not a portable convention, so don't scan for it and don't assume it exists:
   - Check the current working directory directly: does it (or an ancestor) have a `.git` folder/remote (`git rev-parse --show-toplevel`), and does `git status` show staged changes there? If so, that's the theme directory - use it as-is, no further searching, no `cd`.
   - If the cwd isn't a git repo, don't stop to ask and don't go hunting elsewhere in the workspace for it (a broad `Glob`/`ls` scan can be truncated/sorted by mtime and miss the real folder, as happened on the first Quickstart run) - just proceed on the assumption the user is already sitting in the right place.
   - If the user then tells you the actual path (as happened on that same run), use it directly - don't re-derive it from the mapping table.
5. As soon as the store handle is known, hand the user the dev-server command - no running-process check first, no `cd`, regardless of whether the theme directory found in step 4 differs from the current directory. Just give:
   ```
   shopify theme dev --store=<store-handle>
   ```
   as its own fenced code block so it's a single click to copy-paste into whatever terminal/field the user runs it in. It genuinely doesn't matter whether a server is already running or not - never check, and never prepend a `cd`.
   - Right after handing over the command, use `AskUserQuestion` with a Yes/No-style option to ask **"Is development complete on this task and are the changes staged?"** ("Yes - move to QA" / "Not yet") instead of a plain text question - a one-click confirmation is faster for the user than typing a reply. Don't poll for the answer, and don't start a second copy via Claude's own background shell as a substitute for the user's own terminal.
6. Branch on that answer:
   - **Not yet** -> tell the user the environment is ready and **stop here**. Don't proceed into Phase B until they come back and confirm - either by re-invoking `/lets-work` or answering this question `Yes` on a later run.
   - **Yes** -> don't stop the skill - continue straight into Phase B in this same invocation, picking up with its step 1 (re-confirm the active task and branch, then handle staged changes).

## Phase B - Ship for QA

1. Re-confirm the active task and the current branch (`git branch --show-current`, `git status`).
2. If there are uncommitted changes:
   - Ask whether to stage them if they aren't staged yet - never `git add` beyond what's explicitly asked.
   - Draft the commit message using the `git-commit-message` skill (or `shared-tools:git-commit-message` where that's what the repo has enabled) against `git diff --staged`, using the active task as the task-context input.
   - Show the drafted message and **ask for explicit yes/no before running `git commit`**. Re-running `/lets-work` is not itself authorization to commit - the standing "never commit without being asked" rule applies every time, no exceptions for this being a second invocation.
3. Push the branch (`git push -u origin <branch>` first time, `git push` after). Never force-push; if one is genuinely needed, stop and ask.
4. Open the PR: `gh pr create --base <integration branch> --head <branch>`. Get the correct base branch from the repo's own conventions (its CLAUDE.md / git-workflow docs - e.g. shopify-ruti ships from `staging`, not `main`) rather than assuming `main`; ask if the repo has no documented convention. If a PR already exists for this branch, reuse its URL instead of erroring.
5. Add the PR link as a comment on the active task via `add_comment` - do this directly, don't ask the user to do it first. Post it as a plain comment in the exact format `PR: <link>`, nothing else - no mention of the GitHub app card or attaching it themselves. **Only post this once per task**: before adding it, check the task's existing comments (`get_task_stories` or the comments already fetched this session) for a prior `PR:` comment - if one's already there (e.g. this is a later push/re-run of Phase B for the same task, not a first PR), skip posting again rather than adding a duplicate.
6. Ask: "Is this a Retainer or a Project?" via `AskUserQuestion` first - this one has to stay its own round trip since it gates which follow-up question(s) even apply. Then branch:

   - **Retainer** -> in the **same** `AskUserQuestion` call, ask both of the following as separate questions (they don't depend on each other's answer, so there's no reason to ask them one after another): "How should the QA preview theme be created?" and "Who should be assigned for FQA and DQA?" (options for the latter as in step 5 below). Offer for the first:
     - **GitHub-connected theme** - stays in sync with this branch and maps to the PR for later merge. Creating this connection is an OAuth-consent Admin UI action with no CLI or Admin API equivalent (confirmed against `shopify theme push/list/open --help`, and against the Admin GraphQL schema - `themeCreate`'s only input is `source: URL!`, a ZIP/staged-upload URL, never a git/GitHub reference). The user must create it manually.
     - **Manual theme (Shopify CLI)** - a disposable unpublished preview theme Claude pushes directly via `shopify theme push --unpublished`. Fully automatable, but it's a one-time snapshot: it will NOT stay in sync with the branch, and will go stale if more commits are pushed after it's created. Say this plainly when offering the option.
     Retain the QA-assignee answer from this same call for use in step 5 below - don't ask it again later.
   - **Project** -> skip the theme-creation question, but still ask "Who should be assigned for FQA and DQA?" here (same options as step 5 below) in this same `AskUserQuestion` call, then run `shopify theme list --store=<store-handle> --json`, find the store's staging theme, and use its `preview_url` directly as the QA preview URL - no push, no GitHub-connect walkthrough, no resource-aware URL-building. Go straight to "Both paths converge here" below with that link.

### If GitHub-connected theme

1. Ask the user to log into the resolved Shopify store if authentication is needed.
2. Tell the user to: Shopify Admin -> Online Store -> Themes -> "Add theme" -> "Connect from GitHub" -> select this repo and the branch just pushed. Propose a name - `BB - DEV - <dev initials> - <task name>`, where `<task name>` drops the leading bracketed store/project prefix entirely (not just the bracket characters) - e.g. `[Quickstart] - PDP TEST` becomes just `PDP TEST`, `[Ruti] - Utility Pages` becomes just `Utility Pages`. Let the user confirm or adjust it before they create it. Wait for the user to confirm it's done - don't poll for it.
   - **If the store's theme limit is reached** (Shopify Admin blocks adding a new theme), tell the user to go into the store's Admin -> Online Store -> Themes themselves and delete a theme to free up a slot, then re-run `/lets-work` to pick this back up. **Never delete a theme via the CLI/Admin API on the user's behalf** - which theme is safe to remove is their call, not something to infer or automate. Don't offer an `AskUserQuestion` checkbox here - just stop the skill at this point and wait for the next invocation.
3. Once the user confirms the theme is created and they've opened its preview, ask them directly (plain text - this is arbitrary pasted input, not a small fixed choice, so `AskUserQuestion`'s clickable-option pattern doesn't fit here) to paste the theme's preview link. **Use that pasted link as-is as the QA preview URL** - skip the resource-aware URL-building logic in step 4 below entirely for this path; there's no theme ID to build from since Claude never looked one up, and the user-supplied link is already the real thing.

### If Manual theme (Shopify CLI)

1. Determine the theme name using the same convention: `BB - DEV - <dev initials> - <task name>`, where `<task name>` drops the leading bracketed store/project prefix entirely (not just the bracket characters) - e.g. `[Quickstart] - PDP TEST` becomes just `PDP TEST`.
2. Run `shopify theme push --unpublished --theme="<name>" --path="<resolved theme directory>" --store=<store> --json` from Claude's own shell. **Always pass an explicit `--path` pointing at the resolved theme directory from Phase A step 4** - never rely on an earlier `cd` still being in effect. Confirmed the hard way: Bash/PowerShell tool calls in this environment do not persist a working-directory change between invocations (each call resets to the workspace root), so a first attempt without `--path` silently ran from the workspace root, found no real theme structure, and partially uploaded/attempted to delete core files (`layout/theme.liquid`, `config/settings_schema.json`, `templates/gift_card.liquid`) on the newly created theme before erroring. Re-running the same command with the correct `--path` (targeting the same returned theme ID) fixed it.
3. Parse `theme.id` and `theme.preview_url` from the returned JSON.
4. Remind the user this preview theme is a one-time snapshot, not connected to the branch - if they push more commits later and want the QA preview to reflect them, this command needs to be re-run.

### Both paths converge here

4. Build the QA preview URL:
   - **Project path**: already have the staging theme's `preview_url` from step 6 above - use it as-is, skip everything below.
   - **Retainer / GitHub-connected path**: already have the real preview link the user pasted in step 3 above - use it as-is, skip everything below.
   - **Retainer / Manual-CLI path**: build a resource-aware URL from the theme ID just obtained:
     - Determine the touched template(s) from the diff (`templates/page.<suffix>.json` -> page, `templates/product.<suffix>.json` -> product, `templates/collection.<suffix>.json` -> collection). A global/site-wide change with no specific template goes straight to the fallback below.
     - **Existing template, just edited**: find a real resource already assigned to it via Admin GraphQL (`pages`/`products`/`collections` queried by `template_suffix`). If more than one plausible candidate comes back, ask rather than picking one.
     - **Brand-new template, no resource assigned yet**: query for any real resource of that type and force-render the new template with `&view=<suffix>`.
     - **Fallback**: no applicable template, or the lookup returns nothing usable - use the theme's own `preview_url` from the push JSON (bare `?preview_theme_id=<id>`) and say plainly it's the generic homepage, not page-specific.
     - Always use the theme ID just obtained this run - never reuse an ID from an earlier session or a different task.
5. Who to assign for FQA and DQA - fixed options (role is determined by which list a name is picked from, not stated separately):
   - **FQA**: Anjum, Majid, Khurram, Ashhal, Fazeela, Sidra
   - **DQA**: Grace, Olena
   This was already asked as part of step 6's combined `AskUserQuestion` call above - use that answer here rather than asking again. Once picked, still look up each chosen person's real gid live (`search_objects(resource_type="user")` or `get_users`) rather than hardcoding one - never invent a gid even for a name on this fixed list. If the user names someone outside these two lists (typed as free text, e.g. via the question's "Other" option, or stated outright), just look up that person's gid and assign them - no separate yes/no confirmation needed, since naming a specific person is already an explicit, unambiguous instruction.
6. Create the QA subtask(s) - only for the role(s) actually covered by names given:
   - Parent: the active task's gid.
   - Name: `FQA - <active task name>` / `DQA - <active task name>`.
   - Confirm live (don't assume from another client's setup) that the **B&B Team Resource Planning** project and its **Preview Link** and **Status** custom fields exist and get their real gids/enum options via `get_project` before using them - a gid confirmed for one client isn't guaranteed to be right for another.
   - **Create the subtask assigned to the project first** (project = B&B Team Resource Planning, due date = today, Preview Link = the URL from step 4, assignee = the looked-up gid) - **do not set Status in this same call**. Confirmed the hard way: adding a task to this project resets its Status back to "New Request" regardless of what Status was passed in the same create call, so setting it up front gets silently clobbered.
   - **Then, in a separate follow-up update**, set Status = QA on the subtask that now exists in the project.
   - Add a comment "Please Proceed with QA [assigne-name]" via `add_comment`.
7. Update the active task itself: Status -> QA and Preview Link -> the same URL, **but only if the active task's own project actually has those fields** - check via `get_project` rather than assuming every client project mirrors B&B Team Resource Planning's schema; if it doesn't, say so instead of silently skipping or inventing a field.
8. Report back: branch, PR link, theme name and preview URL, and a link to each subtask created. Don't mark anything complete yourself - that's QA's call.

## Constraints carried over from prior work in this workspace

- **Never invent an Asana project/custom-field gid.** Fetch it live via `get_project`/`search_objects` before using it, even when a similar one is already documented for another client (learned building `add-to-qa-retainer` for shopify-ruti - its gids are Ruti-specific, not assumed portable).
- **Never invent a project/store/folder mapping** not in the table above or in `client-theme/` - ask if something doesn't match.
- **Never commit without an explicit yes/no**, even on a Phase B re-run.
- **Connecting a Shopify theme to a GitHub branch is manual, Admin-UI-only** - confirmed via `shopify theme --help`/`theme push/list/open --help` AND via the Admin GraphQL schema (`themeCreate`'s only source input is `source: URL!`, a ZIP/staged-upload URL, never a git/GitHub reference - there is no mutation or argument anywhere for a GitHub link). Always pause for user confirmation; never poll for it. Separately, **pushing a disposable unpublished preview theme IS scriptable** via `shopify theme push --unpublished --theme="<name>" --path="<dir>" --json` - this is the "Manual theme (Shopify CLI)" option, a one-time snapshot only, not a substitute for the real GitHub connection.
- **Attaching a PR to Asana's GitHub app card is manual, UI-only** - no MCP tool exists for it (checked the full Asana tool list before assuming otherwise). The PR-link comment itself stays a plain `PR: <link>` with no mention of the app card.
- **QA assignee options are a fixed list** (FQA: Anjum, Majid, Khurram, Ashhal, Fazeela, Sidra; DQA: Grace, Olena) - role is implied by which list a name comes from. Still never hardcode a person's gid - look it up live even for a name on this list.
- **Never force-push**, and always check the repo's actual documented base branch before opening a PR - some client repos ship from `staging`, not `main`.
- **QA subtask creation must add the task to B&B Team Resource Planning *before* setting Status to QA, never in the same call.** Adding a task to this project resets its Status back to "New Request" - if Status = QA is set in the same create call as the project assignment, the project add silently overwrites it back to New Request. Always create the subtask in the project first, then a separate follow-up update sets Status = QA.
- **Claude has no tool that can drive the visible VS Code integrated terminal panel** - confirmed by searching the full available tool set. `Bash`/`PowerShell` calls (including `run_in_background`) run in Claude's own separate managed shell, not that panel. Starting `shopify theme dev` so it's visible there is always a manual handoff: give the user the command, let them run it, wait for their confirmation - never substitute Claude's own background shell as a silent stand-in for "it's running."
