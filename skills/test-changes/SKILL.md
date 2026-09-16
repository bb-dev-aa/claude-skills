---
name: test-changes
description: Generates comprehensive Playwright test cases from staged Shopify theme code changes and Asana task context, then runs them against the live preview theme. Reads staged git changes (*.liquid, *.js, *.css, *.scss, *.json), fetches Asana task details for requirements, extracts design context from linked Figma (optional), dynamically detects the running theme via `shopify theme info`, and creates test cases covering happy paths, error scenarios, responsive behavior, accessibility, and regression testing. Tests run immediately and results post to chat with pass/fail counts, failed test details, and links to HTML reports organized by client and task.
---

# test-changes — Playwright Test Generation & Execution for Shopify Themes

## Purpose

Instead of manually testing every code change (happy path, errors, responsive, accessibility, regression), the skill reads your staged Shopify theme changes and Asana task context, generates comprehensive Playwright test cases automatically, runs them against your live preview theme, and reports pass/fail results in chat.

**Workflow:**
```
User: /test-changes
    ↓
[Skill] Select Asana task
    ↓
[Skill] Read staged git changes + Asana context + Figma (optional)
    ↓
[Skill] Analyze: "What changed? What should it do?"
    ↓
[Skill] Generate Playwright tests (happy path, errors, responsive, a11y, regression)
    ↓
[Skill] Run tests against live preview theme
    ↓
[Chat] Report: X passed, Y failed, link to results
```

---

## Required Connections

This skill needs three MCP connectors available in the current session:
- **Asana** — Read tasks, titles, descriptions, comments, attachments
- **Git** (via bash) — Read staged changes
- **Figma** (optional) — Read design context from linked files

**Before proceeding, confirm these are available.** If Asana or Git is missing, stop and tell the user.

---

## Invocation

```
/test-changes
```

No arguments — the skill guides the user through task selection and change detection.

---

## Execution Flow

### **Phase 1: Task & Context Gathering**

#### Step 1 — Fetch recent Asana tasks
- Call `get_me` to confirm Asana connection
- Call `get_my_tasks(completed_since="now")` to list recent tasks
- **Present a list to the user:** "Select the task you're testing"
  - Show task title (with `[Client]` prefix), due date, status
  - Offer: "Pick one" / "Enter task URL/ID" / "Cancel"
- Wait for user selection

#### Step 2 — Fetch full task context
- Get selected task's full record: title, description, comments, attachments, custom fields
- Extract **client name** from title pattern `[Client]`: 
  - `[Ruti] Sticky Header – Fix Mobile` → client = `Ruti` → normalize to `ruti`
  - `[GIR] PDP A/B Test – Sizing Verbiage` → client = `GIR` → normalize to `gir`
- Extract **task name** (omit `[Client]` prefix):
  - `[Ruti] Sticky Header – Fix Mobile` → `sticky-header-fix-mobile` (slugified, lowercase, kebab-case)
- Check for **Figma links** in description or attachments

#### Step 3 — Read staged changes
- Run `git diff --staged` to get file changes
- Filter for Shopify theme files: `*.liquid`, `*.js`, `*.css`, `*.scss`, `*.json`
- Parse: which components/sections were modified?
- **If no staged changes:** stop and tell user "No staged changes detected. Stage your changes first."

#### Step 4 — Detect running dev server
- Run `shopify theme info`
- Parse output to extract:
  - **Store:** `tempruti.myshopify.com`
  - **Theme ID:** `189756997999`
- **Construct preview URL:**
  ```
  https://tempruti.myshopify.com/?preview_theme_id=189756997999
  ```
- **If command fails:** tell user "Dev server not running. Start it with `shopify theme dev --store=<handle>`"

#### Step 5 — Read Figma context (optional)
- If Figma link found in task, call Figma MCP to read design context
- Extract: component structure, interactions, responsive breakpoints
- If Figma link invalid or unavailable, skip silently — proceed without design context

---

### **Phase 2: Analysis**

#### Step 6 — Understand the changes
Synthesize all gathered context:
- **What changed?** (from git diff)
- **Why?** (from Asana task description + comments)
- **How should it work?** (from Asana acceptance criteria + Figma design)
- **What could break?** (identify adjacent features / regression areas)

Determine test scope:
- ✅ Happy path: intended behavior works
- ✅ Error scenarios: validation, edge cases, missing data
- ✅ Responsive: mobile (320px), tablet (768px), desktop (1024px+)
- ✅ Accessibility: WCAG AA standards, keyboard nav, screen reader compatibility
- ✅ Regression: existing features in touched files still work

---

### **Phase 3: Generate Tests**

#### Step 7 — Create Playwright test file

Generate a `.spec.ts` test file with:

**Structure:**
```typescript
import { test, expect } from '@playwright/test';

// [HAPPY PATH TESTS]
test.describe('Component/Section Name - Happy Path', () => {
  test('should load and render correctly', async ({ page }) => {
    // ...
  });
  
  test('should handle user interaction as intended', async ({ page }) => {
    // ...
  });
});

// [ERROR SCENARIO TESTS]
test.describe('Component/Section Name - Error Scenarios', () => {
  test('should handle missing data gracefully', async ({ page }) => {
    // ...
  });
  
  test('should show validation on invalid input', async ({ page }) => {
    // ...
  });
});

// [RESPONSIVE TESTS]
test.describe('Component/Section Name - Responsive', () => {
  test.use({ viewport: { width: 320, height: 667 } });
  test('should be usable on mobile', async ({ page }) => {
    // ...
  });
  
  test.use({ viewport: { width: 1024, height: 768 } });
  test('should display correctly on desktop', async ({ page }) => {
    // ...
  });
});

// [ACCESSIBILITY TESTS]
test.describe('Component/Section Name - Accessibility', () => {
  test('should be keyboard navigable', async ({ page }) => {
    // ...
  });
  
  test('should have proper ARIA labels', async ({ page }) => {
    // ...
  });
});

// [REGRESSION TESTS]
test.describe('Adjacent Features - Regression', () => {
  test('should not break existing feature X', async ({ page }) => {
    // ...
  });
});
```

**Test naming:** Clear, descriptive names explaining what's being tested.

**Test content:** Use Playwright best practices:
- Locate elements via accessible selectors (role, label, placeholder, test ID)
- Assert on visible behavior, not implementation details
- Use explicit waits for async operations
- Clean up state between tests if needed

**Output:** Single `.spec.ts` file as a **code artifact** for the user to review before running.

---

### **Phase 4: Execute Tests**

#### Step 8 — Prepare execution environment

Set up environment variables and run tests:

**Task slug generation:**
- Input: `sticky-header-fix-mobile` (extracted task name)
- Client: `ruti` (extracted from `[Ruti]`)
- Timestamp: `2026-09-03-14-30-45`
- Results folder: `workspace/playwright/test-results/ruti/ruti-sticky-header-fix-mobile-2026-09-03-14-30-45/`

**Environment variables to pass:**
```bash
CLIENT_NAME=ruti
TASK_SLUG=sticky-header-fix-mobile
BASE_URL=https://tempruti.myshopify.com/?preview_theme_id=189756997999
TEST_DIR=workspace/client-theme/Ruti/tests/playwright
```

**Find test directory:**
- Check if `workspace/client-theme/[Client]/tests/playwright/` exists
  - ✅ Exists → use it
  - ❌ Doesn't exist → create folder structure
- Place generated test file here

#### Step 9 — Run Playwright tests

Execute:
```bash
CLIENT_NAME=ruti \
TASK_SLUG=sticky-header-fix-mobile \
BASE_URL=https://tempruti.myshopify.com/?preview_theme_id=189756997999 \
TEST_DIR=workspace/client-theme/Ruti/tests/playwright \
npx playwright test --config=workspace/playwright/playwright.config.ts
```

**Capture results:**
- ✅ Pass count
- ❌ Fail count
- Test names that failed
- Error messages / assertion failures
- Screenshots / videos (for failures)
- HTML report generated at: `workspace/playwright/test-results/ruti/ruti-sticky-header-fix-mobile-2026-09-03-14-30-45/index.html`

---

### **Phase 5: Report Results**

#### Step 10 — Post summary to chat

Format:
```
✅ Tests Generated & Executed

Task: [Ruti] Sticky Header – Fix Mobile Scroll
Client: Ruti
Preview: https://tempruti.myshopify.com/?preview_theme_id=189756997999
Test Time: 45 seconds

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RESULTS
✅ Passed: 18 tests
❌ Failed: 2 tests

FAILED TESTS:
  ❌ Gift Card Preview – Responsive – should be usable on mobile
  ❌ Gift Card Preview – Accessibility – should be keyboard navigable

ERROR DETAILS:
  • Gift Card Preview – Responsive: Expected "visibility: visible" but got "visibility: hidden"
  • Gift Card Preview – Accessibility: Tab order is incorrect; button not reachable

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📊 VIEW FULL REPORT:
workspace/playwright/test-results/ruti/ruti-sticky-header-fix-mobile-2026-09-03-14-30-45/index.html

📁 TEST FILE:
workspace/client-theme/Ruti/tests/playwright/ruti-sticky-header-fix-mobile.spec.ts
```

If all tests pass:
```
✅ All 20 tests passed! Ready for QA.
```

If all tests fail:
```
❌ All tests failed. Check the generated test file — may need adjustments.
```

---

## Edge Cases & Handling

| Situation | Action |
|---|---|
| No Asana connection | Stop, tell user Asana MCP is required |
| No git repo / no staged changes | Stop, tell user to stage changes |
| Dev server not running | Stop, tell user to run `shopify theme dev --store=<handle>` |
| Figma link invalid | Skip silently, proceed without design context |
| Client folder doesn't exist | Create `client-theme/[Client]/tests/playwright/` structure |
| Test execution fails (timeout, crash) | Return error log + suggest debugging steps |
| Same client, multiple runs | Each run gets unique timestamp → separate result folder |
| User cancels task selection | Stop gracefully, tell user to re-run `/test-changes` |

---

## Configuration Files

### `workspace/playwright/playwright.config.ts`
Already created by user. Uses env vars for dynamic configuration:
- `BASE_URL` — from `shopify theme info`
- `CLIENT_NAME` — from Asana task `[Client]`
- `TASK_SLUG` — from task name (slugified)
- `TEST_DIR` — from detected client folder

### Results Folder Structure
```
workspace/playwright/test-results/
├── ruti/
│   ├── ruti-sticky-header-fix-mobile-2026-09-03-14-30-45/
│   │   ├── index.html
│   │   ├── data/
│   │   └── videos/
│   └── ruti-utility-pages-2026-09-04-10-15-22/
│
├── gir/
│   └── gir-pdp-ab-test-2026-09-03-15-45-22/
│       ├── index.html
│       ├── data/
│       └── videos/
│
└── [all other clients same structure]
```

---

## Implementation Notes

- **Playwright runs headless by default** — tests execute in background, results stream to chat
- **Single worker mode** — tests run sequentially (safer for Shopify preview themes)
- **Explicit waits** — don't assume elements are ready; use `waitForSelector` or role queries
- **Page Objects optional** — if client already has test helpers/fixtures, use them; generate tests to match existing patterns
- **No auth required** — preview themes are publicly accessible; if auth is needed, surface it as an error
- **Screenshots on failure** — Playwright captures failures automatically, linked in results
- **HTML report** — user can open the report in browser for full details, videos, traces

---

## Quick Reference

| Command | Purpose |
|---|---|
| `/test-changes` | Start the skill, select task, generate & run tests |
| View results | Open `workspace/playwright/test-results/[client]/[task]/index.html` |
| Re-run same tests | `npx playwright test --config=workspace/playwright/playwright.config.ts` |
| Run specific test | `npx playwright test --grep "test name"` |

---

## Learnings Log

*Entries added after real usage:*

- *[TBD: First real run]*