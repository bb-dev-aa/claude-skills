Base directory for this skill: `.claude/skills/test-changes`

# test-changes: Simple E2E Testing for Shopify Theme Changes

Generates and runs **one Playwright spec** for the currently staged Shopify theme changes, checking four things only:

1. **Functional** — the changed section(s) render and behave correctly.
2. **Breakpoints** — no overflow/clipped content/tiny tap targets, at mobile and desktop widths.
3. **Accessibility** — no serious/critical axe violations.
4. **Figma match** — key measurements (padding, gaps, font size/weight, color) match the Figma design.

Nothing else. No unit tests, no Lighthouse, no pixel/screenshot diffing, no Theme Editor simulation, no shared support library — the generated spec is self-contained and just imports `@playwright/test` and `@axe-core/playwright` directly.

## Where things live

Everything this skill reads or writes is under `playwright/` at the root of this workspace (`Workspace\playwright\`) — never inside a `client-theme/<Client>/...` repo:

```
playwright/
  playwright.config.ts
  test-results/<client>/<client>-<task-slug>-<timestamp>/
    <task-slug>.spec.ts
    results.json           # Playwright's own output (json reporter only — no html report, no traces/videos)
```

`node_modules`/`package.json` there already have `@playwright/test` and `@axe-core/playwright` installed — don't reinstall or change them. If a run ever needs a package that isn't there, install just that one package, nothing more.

## Steps

1. **Find the Asana task.** `get_me` → `get_my_tasks`. If more than one plausible task, `AskUserQuestion` to pick (a real multiple-choice pick — fixed options are fine here). Read it in full: `get_task` with comments/subtasks, plus `get_attachments`. Client = the first `[Client]` token, lowercased, path-safe (`[GIR]` → `gir`). Task slug = the rest, kebab-case.

2. **Figma link.** Look for one in the task's description, comments, and attachments. If there isn't one, ask for it with `AskUserQuestion`: phrase the question to tell the user to paste it in the **Other** field. Options otherwise: "Skip design check" and "Cancel".

3. **Staged changes.** `git diff --staged --name-status` and `git diff --staged` (in the relevant client repo). Keep `*.liquid, *.js, *.css, *.scss, *.json`. If nothing is staged, stop and say so.

4. **Preview link.** Ask with `AskUserQuestion`, free text: tell the user to paste the Shopify Theme Editor "Copy link" URL in the **Other** field. Option: "Cancel". Use the reply verbatim — never ask a follow-up to double check it, and never verify it yourself with curl/WebFetch (Shopify bot-filters those and silently serves the published theme — only a real browser, i.e. Playwright, is trustworthy here).

5. **Figma measurements.** With a Figma link, use `get_metadata` to find the frame(s) for the changed section at each breakpoint (mobile/desktop), then `get_design_context` / `get_variable_defs` on its key layers. Pull only what the diff's own CSS actually controls — padding, gaps, max-width, font-size/weight, color — and only where the mapping to a selector in the diff is unambiguous. Skip anything set by a shared/global class outside the diff, or where Figma's frame geometry doesn't cleanly correspond to a real CSS property.

6. **Generate the spec.** One file: `playwright/test-results/<client>/<client>-<task-slug>-<timestamp>/<task-slug>.spec.ts`. Structure:
   - `test.describe('<Section>: Functional')` — renders correctly, acceptance criteria from the task if any, regression scoped to the changed file(s) only (never header/footer/unrelated sections).
   - `test.describe('<Section>: Breakpoints')` — runs across the config's four fixed projects: `large` (1920x1080), `desktop` (1512x923), `tablet` (834x1112), `mobile` (iPhone 14 Pro). No horizontal scroll, section stays inside the viewport, tap targets ≥ 24px. Never use two `test.use({ viewport })` calls in the same describe — the last one silently wins; use Playwright projects instead.
   - `test.describe('<Section>: Accessibility')` — `AxeBuilder` scoped to the section, fail only on serious/critical.
   - `test.describe('<Section>: Figma match')` — for each measurement from Step 5, assert the live computed style against the Figma value, with a plain-English failure message, e.g.:
     ```ts
     const expected = 60, actual = parseFloat(await el.evaluate(n => getComputedStyle(n).paddingTop));
     expect(Math.abs(actual - expected), `Padding is inconsistent: expected ${expected}px, got ${actual}px`).toBeLessThanOrEqual(1);
     ```
     Every mismatch should read as a plain sentence like this, not a raw selector/property dump.
   - Selectors: use the `#id`/`.class` names actually present in the diff (tag selectors like `header` miss custom elements). Navigate with `page.goto()` on the preview URL, preserving its query string (`preview_theme_id` etc. — don't drop it by joining a bare path).

   Show the generated spec to the user, then run it without waiting for further confirmation.

7. **Run it.**
   ```bash
   cd playwright
   TEST_DIR=<abs run folder> BASE_URL=<preview link verbatim> \
     npx playwright test --config=playwright.config.ts --retries=2 --timeout=45000
   ```
   A failure that repeats identically on every retry is real — read the error before calling it flaky.

8. **Report in chat**, plainly:
   ```
   Functional      ✅ 3 passed
   Breakpoints     ✅ 5 passed  ❌ 1 failed (375px: content overflows viewport)
   Accessibility   ✅ 0 serious/critical
   Figma match     ❌ Padding is inconsistent: expected 60px, got 40px (desktop, .case-study-list)

   Results: playwright/test-results/<run folder>/results.json
   Spec:    playwright/test-results/<run folder>/<task-slug>.spec.ts
   ```
   All passed → "✅ All checks passed."

## A few things not to do

- Don't grep the theme for where a new section is placed, and don't ask — the preview link is what decides what gets tested.
- Don't touch anything outside the changed file(s) (no header/footer/unrelated-section regression).
- Don't write any generated file outside `playwright/` — never into a client's own repo.
- Don't reinstall or prune `playwright/node_modules`/`package.json` as part of a normal run.
