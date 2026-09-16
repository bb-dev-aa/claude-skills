# Claude Skills — Shopify Theme Dev Workflow

A small set of [Claude Code](https://claude.com/claude-code) [Agent Skills](https://code.claude.com/docs/en/skills) built for day-to-day Shopify Online Store 2.0 theme development. Each skill automates one recurring, time-consuming part of the workflow — picking up and shipping a task, capturing what you learned along the way, and testing what you changed — so that work stays in the editor instead of bouncing across Asana, the terminal, GitHub, and Shopify Admin by hand.

They were built and refined against real client theme work over several iterations, not written speculatively — see each skill's own "Learnings log" for the dated history of what changed and why.

## Why these exist

Theme development for multiple clients tends to accumulate the same three time sinks:

- **Context switching** — every task means re-figuring out which store, which repo, which branch, and re-doing the same handful of terminal/GitHub/Shopify steps to get a task from "picked up" to "in QA."
- **Knowledge loss** — the real reasoning behind a fix (root cause, a Shopify platform quirk, a pattern that worked) usually lives only in a PR diff or a Slack message, and gets rediscovered from scratch by the next person who hits it.
- **Manual QA overhead** — testing every change's happy path, error states, responsive behavior, accessibility, and regressions by hand takes hours and still misses things.

Each skill below targets exactly one of these.

## Skills

| Skill | Solves | Requires |
|---|---|---|
| [`/lets-work`](skills/lets-work/SKILL.md) | Task pickup → dev → QA handoff | Asana MCP · Shopify CLI · GitHub CLI (`gh`) |
| [`/devdocs`](skills/devdocs/SKILL.md) | Capturing & reloading engineering knowledge | Asana MCP · GitHub · Google Drive |
| [`/test-changes`](skills/test-changes/SKILL.md) | Generating & running tests against changes | Asana MCP · Git · Shopify CLI · Playwright · Figma MCP (optional) |

### [`/lets-work`](skills/lets-work/SKILL.md) — session orchestrator

The single entry point for "what am I working on right now." It tracks the active task for the whole session and knows which phase it's in, so re-running it mid-task resumes correctly instead of restarting.

- **Start:** loads your open Asana task (or a named one), resolves which client project it belongs to — store handle + local theme folder — moves the task to **"In Dev,"** and hands you the exact `shopify theme dev --store=<handle>` command to run.
- **Ship:** once you confirm the change is done and staged, it drafts a commit message, pushes the branch, opens a PR against the repo's own correct base branch, and links the PR back on the Asana task.
- **QA handoff:** stands up a QA preview (a real GitHub-connected theme, a disposable CLI-pushed one, or the store's existing staging theme, depending on engagement type), then files FQA/DQA subtasks with the preview link attached and updates the task's status.

### [`/devdocs`](skills/devdocs/SKILL.md) — knowledge capture & recall

Turns a completed task and its linked PR into durable, searchable documentation instead of letting it evaporate.

- **Capture:** reads a completed task + its PR, classifies what was learned (**Bug Fix, New Pattern, Shopify Limitation, Best Practice,** or **Discovered Architecture**), and writes it into that client's changelog (and architecture doc, if relevant) — deduplicated so re-runs update rather than duplicate entries.
- **Load (`get-context`):** pulls a client's existing docs back into the session *before* new work starts, and actively cross-checks new work against known fixes/limitations as you go.
- **Shared promotion:** genuinely reusable knowledge only gets promoted to cross-client shared docs through an explicit, separate step — never automatically, so shared docs stay curated rather than a dumping ground.

### [`/test-changes`](skills/test-changes/SKILL.md) — generate & run tests

Closes the loop on a change by turning "did I break anything?" into an automated check instead of a manual pass.

- Reads your staged theme file changes together with the relevant Asana task (and a linked Figma design, if any) to understand what changed and what it's supposed to do.
- Generates a Playwright test file covering **happy path, error scenarios, responsive breakpoints, accessibility, and regressions** on anything else the change touched.
- Runs it against your live preview theme and reports pass/fail counts, failure details, and a link to the full HTML report — organized by client and task.

## How they fit together

The three skills cover sequential stages of the same task, and are meant to be used in order: `/lets-work` to start the task and get it into a dev environment, `/test-changes` to verify the change before shipping, then back to `/lets-work` to ship and hand off for QA, and finally `/devdocs` to capture whatever was learned once it's done.

## Setup

### Option A — install as a plugin (recommended)

This repo is a Claude Code plugin marketplace (`.claude-plugin/marketplace.json`), so it can be installed directly without copying files:

```bash
/plugin marketplace add <owner>/claude-skills
/plugin install shopify-theme-skills@claude-skills
```

Claude Code fetches the `skills/` folder and makes `/lets-work`, `/devdocs`, and `/test-changes` available automatically. Re-running `/plugin marketplace add` after a push picks up updates.

### Option B — copy manually

1. Copy the `skills/` folder into your own workspace's `.claude/skills/` directory (or a specific project's).
2. Connect whichever MCP servers the skills you're using need — Asana, Google Drive, Figma, GitHub — inside Claude Code.
3. Run the slash command for the skill you want, e.g. `/lets-work`.

Either way: each `SKILL.md` documents its own required connections, configuration (e.g. the Asana project name and QA roster, the Google Drive doc root), and edge case handling — see the linked files above for full detail. These are written for a specific team's Asana/Drive setup as an example; adapt the fixed names/tables inside each `SKILL.md` (client-to-store mappings, QA assignee lists, Drive folder IDs) to your own before using them elsewhere.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [Shopify CLI](https://shopify.dev/docs/api/shopify-cli)
- [GitHub CLI](https://cli.github.com/) (`gh`)
- MCP connectors for Asana, Google Drive, and (optionally) Figma
- [Playwright](https://playwright.dev/) for `/test-changes`
