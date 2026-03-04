
# Task 1: GOV.UK Holiday Entitlement Calculator — Playwright Test Suite

An end-to-end (E2E) test suite built with [Playwright](https://playwright.dev/) that automates and validates the [GOV.UK Holiday Entitlement Calculator](https://www.gov.uk/calculate-your-holiday-entitlement). The suite covers multiple calculation flows and validation/error scenarios, all organised using the **Page Object Model (POM)** pattern.

---

## Project Structure

```
project-root/
├── node_modules/                        # Installed dependencies
│   ├── @playwright/
│   ├── playwright/
│   ├── playwright-core/
│   └── typescript/
├── pages/
│   └── HolidayEntitlementPage.ts        # Page Object — all locators and reusable actions
├── test-results/
│   └── .last-run.json                   # Playwright last run metadata
├── tests/
│   ├── holiday-entitlement.spec.ts      # Happy-path flow tests (Flows A–D)
│   └── holiday-validation.spec.ts       # Negative / validation tests
├── .package-lock.json
├── package-lock.json
├── package.json                         # Project dependencies and scripts
├── playwright.config.ts                 # Playwright configuration
└── README.md                            # Project documentation
```


## What Is Being Tested

The suite tests the multi-step GOV.UK form for calculating statutory holiday entitlement. The form is wizard-style — each step is a separate page — and covers the following user journeys.

**Test case selection was driven primarily by priority.** The happy-path flows cover the most commonly used calculation routes (full year, mid-year start, mid-year leave, and hours-based), as these represent the highest-risk scenarios where a defect would affect the greatest number of users. The validation tests focus on the error conditions most likely to be encountered in practice — empty fields, missing date parts, and out-of-range numeric inputs — rather than attempting exhaustive coverage of every possible invalid state. This prioritisation ensures the most business-critical journeys are protected first, with lower-priority edge cases being candidates for future test additions as time allows.

### Happy-Path Flows (`holiday-entitlement.spec.ts`)

| Test | Scenario | Expected Result |
|------|----------|-----------------|
| **Flow A** | 5 days/week, full leave year | 28 days entitlement |
| **Flow B** | 37.5 hours/week, full leave year, 5 days/week | 210 hours entitlement |
| **Flow C** | 5 days/week, starting mid-year (15 July 2024) | 16.5 days entitlement |
| **Flow D** | 5 days/week, leaving mid-year (15 Nov 2024) | 22.2 days entitlement |

All flows use a shared leave year start date of **01 February 2024**.

### Validation Tests (`holiday-validation.spec.ts`)

| Test | Scenario |
|------|----------|
| All date fields empty | Error summary is shown |
| Missing year in date | Error summary is shown |
| Invalid date (31 Feb 2024) | Error summary is shown |
| Days/week field left empty | Error summary is shown |
| Days/week set to 0 | Error summary is shown |
| Hours/week set to a negative number | Error summary is shown |

---

Just drop this updated section back into the full README in place of the existing one. Let me know if you'd like the full document regenerated as a download with all the latest changes combined.

## Playwright Features Used

### 1. UI Mode

This project uses **Playwright UI Mode** as the primary way to run and debug tests during development. UI Mode opens an interactive browser-based interface that gives full visibility into every step of a test run.

Launch it with:

```bash
npx playwright test --ui
```

Key capabilities UI Mode provides for this project:

- **Test explorer** — browse and run individual tests or entire `test.describe` groups (e.g. run only the validation tests) without touching the command line.
- **Step-by-step timeline** — each action (`goto`, `click`, `fill`, `check`, `expect`) appears as a named step in a timeline. Hovering over a step highlights the exact DOM element that was targeted at that moment.
- **DOM snapshots** — a before/after snapshot of the page is captured for every action, so you can inspect what the GOV.UK form looked like at any point in the flow without re-running the test.
- **Live watch mode** — changes to spec files or the `HolidayEntitlementPage` Page Object are detected automatically and the affected tests re-run instantly, making the edit-run-debug loop very fast.
- **Trace viewer built in** — the same trace data recorded by `trace: "on-first-retry"` in `playwright.config.ts` is surfaced directly inside UI Mode, so failed tests can be investigated without opening a separate report.
- **Network & console panels** — inspect outgoing requests and browser console output alongside each step, useful for catching unexpected redirects on the GOV.UK multi-step form.

> UI Mode is recommended for local development and exploratory debugging. For CI pipelines, use `npx playwright test` (headless) to keep runs fast and artefact-light.

---

### 2. Locators

The suite uses the full range of Playwright's recommended locator strategies.

**`getByRole()`** — the preferred accessibility-first approach. Finds buttons, radio buttons, headings, and alert regions by ARIA role and accessible name:
```ts
page.getByRole("button", { name: /Start now/i });
page.getByRole("radio", { name: /days worked per week/i });
page.getByRole("heading", { name: /information based on your answers/i });
page.getByRole("alert"); // GOV.UK error summary region
```

**`getByLabel()`** — finds form inputs by their associated `<label>` text. Used for all text inputs:
```ts
page.getByLabel(/^day$/i);
page.getByLabel(/number of days worked per week/i);
```

**`getByText()`** — finds elements by their visible text content. Used to verify calculated results:
```ts
page.getByText(/The statutory (holiday )?entitlement is 28 days/i);
```

**`locator()` with a CSS class** — used as a fallback to target GOV.UK-specific error message elements:
```ts
page.locator(".govuk-error-message").first();
```

---

### 3. Auto-Waiting

Playwright automatically waits for elements to be actionable before interacting with them (visible, enabled, stable in the DOM). The suite relies on this for all `.click()`, `.fill()`, `.check()`, and `expect()` calls, removing the need for manual sleep statements in most places.

---

### 4. Assertions (`expect`)

Playwright's built-in `expect` library provides auto-retrying assertions — they are retried until they pass or a timeout is reached, making them resilient to minor rendering delays.

| Assertion | Purpose |
|-----------|---------|
| `toBeVisible()` | Confirms an element is rendered and visible in the viewport |
| `toContainText()` | Checks that an element's text includes a given string or regex |

---

### 5. `waitForLoadState()`

Called after navigation-triggering button clicks to ensure the browser finishes processing the page transition before the next action runs:

```ts
await this.page.waitForLoadState("domcontentloaded");
```

`"domcontentloaded"` is used rather than `"load"` for speed — it fires once the HTML is parsed, without blocking on external images and stylesheets.

---

### 6. `page.goto()` with `waitUntil`

The initial navigation uses the same `domcontentloaded` strategy for consistency:

```ts
await this.page.goto("https://www.gov.uk/calculate-your-holiday-entitlement", {
  waitUntil: "domcontentloaded",
});
```

---

### 7. `waitForTimeout()`

Used sparingly in `selectEndingPartWayThroughLeaveYear()` to handle a timing edge case where a radio option renders slightly after the DOM-ready event:

```ts
await this.page.waitForTimeout(1000);
```

> **Note:** `waitForTimeout` is a last resort. Prefer `waitForLoadState` or auto-waiting assertions wherever possible.

---

### 8. Form Interactions

| Action | Playwright API |
|--------|---------------|
| Click a button | `locator.click()` |
| Check a radio button | `locator.check()` |
| Fill a text input | `locator.fill(value)` |
| Clear a text input | `locator.fill("")` |

`fill()` always clears the field before typing, making it safe for both initial input and overwriting existing values.

---

### 9. Regular Expressions for Locators

All locator name/label patterns use `RegExp` rather than exact strings, making them resilient to minor copy changes on the GOV.UK page (extra whitespace, capitalisation, optional words):

```ts
{ name: /Start now/i }
{ name: /starting part ?way through (a )?leave year/i }
new RegExp(`The statutory (holiday )?entitlement is ${expectedValue} ${unit}`, "i")
```

---

### 10. `test.describe()` Grouping

Tests are grouped using `test.describe()` blocks, which logically separate happy-path flows from validation tests, allow selective execution with `--grep`, and produce clearer nested output in both the HTML report and UI Mode's test explorer:

```ts
test.describe("GOV.UK Holiday Entitlement - Multiple Flows", () => { ... });
test.describe("GOV.UK Holiday Entitlement - Validation tests", () => { ... });
```

---

### 11. Defensive Visibility Check with `isVisible().catch()`

Used in `clearDaysOrHoursInput()` to probe which input field is currently on-screen without throwing if the locator is absent. This allows one helper method to work across different steps that show either a days or an hours input:

```ts
if (await locator.isVisible().catch(() => false)) {
  await locator.fill("");
  return;
}
```

---

### 12. Playwright Configuration (`playwright.config.ts`)

| Option | Value | Effect |
|--------|-------|--------|
| `testDir` | `./tests` | Where Playwright looks for spec files |
| `headless` | `true` | Runs the browser invisibly — ideal for CI |
| `trace` | `on-first-retry` | Records a trace file only when a test is retried after failure |
| `screenshot` | `only-on-failure` | Captures a screenshot only on failure |
| `video` | `retain-on-failure` | Records video of the session, kept only on failure |

These options balance debugging capability with performance — artefacts are only stored when something goes wrong. When running via UI Mode, trace and snapshot data is always available interactively regardless of these settings.

---

## Running the Tests

### UI Mode (recommended for local development)

```bash
npx playwright test --ui
```

Opens the interactive UI where you can browse, run, filter, and debug any test with full step-by-step snapshots.

### Run All Tests (headless)

```bash
npx playwright test
```

### Run a Specific File

```bash
npx playwright test tests/holiday-entitlement.spec.ts
npx playwright test tests/holiday-validation.spec.ts
```

### Run in Headed Mode

```bash
npx playwright test --headed
```

### View the HTML Report

```bash
npx playwright show-report
```

---

## Design Decisions

- **Page Object Model** keeps test logic separate from DOM interaction logic. Tests read like plain English; the Page Object handles all the Playwright details.
- **UI Mode for development** gives instant visual feedback during test authoring without needing to re-run the full suite each time.
- **Regex locators** are preferred over exact strings to reduce brittleness against minor GOV.UK content changes.
- **`domcontentloaded`** is used consistently as the wait strategy — faster than `networkidle` and sufficient for server-rendered GOV.UK pages.
- **Composite helper methods** (`calcHoursFullYear`, etc.) reduce duplication across tests, while still allowing fully explicit step-by-step tests (Flow A) where readability matters more than brevity.


---

### What I Would Improve Given More Time

**Test data management** is the first thing I'd revisit. Right now, dates and expected values like `28`, `210`, `16.5` are scattered as magic numbers inside the spec files. I'd extract these into a dedicated fixture file or a data-driven structure, so each test case reads as a clear input/output pair and adding new scenarios requires no code changes — just a new data row.

Alongside that, I'd apply **Boundary Value Analysis (BVA)** to drive the test data more rigorously. The current validation tests only probe a handful of invalid inputs, but BVA would give the data selection a systematic basis — for example, testing days worked per week at the exact boundaries of `0`, `1`, `7`, and `8`, or hours per week at `0`, `0.1`, `168`, and `168.1`. This is particularly important for a statutory calculator where off-by-one errors at the boundaries directly translate to incorrect legal entitlements. Combining BVA with a data-driven approach means the boundary cases live in a fixture table rather than being buried in individual test bodies, making coverage gaps immediately visible.

**Environment configuration** is missing entirely. The base URL is hardcoded in `HolidayEntitlementPage.ts`. I'd move it into `playwright.config.ts` using the `baseURL` option, so the suite can target staging or production without touching test code.

**Retry and timeout strategy** deserves more thought. The single `waitForTimeout(1000)` in `selectEndingPartWayThroughLeaveYear()` is a code smell — it's fragile and will either be too short on a slow connection or wastefully slow on a fast one. I'd replace it with a proper `waitFor` condition, polling for the radio button to become visible rather than guessing a fixed delay.

**Cross-browser coverage** isn't configured at all. The config only runs the default browser. I'd add Chromium, Firefox, and WebKit as named projects in `playwright.config.ts`, and potentially add a mobile viewport project to verify the GOV.UK responsive layout behaves correctly on smaller screens.

**CI integration** is absent. I'd add a GitHub Actions (or equivalent) workflow file that runs the suite headlessly on every pull request, uploads the Playwright HTML report as a build artefact, and posts a test summary comment back to the PR. The `trace: "on-first-retry"` config is already primed for this — it just needs the pipeline to use it.

**Reporting** could be richer. Beyond the built-in HTML report, I'd integrate Allure or a similar reporter to produce a historical trend view, which makes it easy to spot flaky tests over time rather than investigating failures in isolation.

**API-layer shortcuts** are worth considering for the mid-year flows. Navigating through five or six form steps to reach the step you actually want to test is slow and couples unrelated tests together — if step two breaks, every test after it fails for the wrong reason. Where GOV.UK supports it, I'd use `page.goto()` to jump directly to a deep URL or seed session state via the API, reserving full end-to-end navigation for the Flow A–D happy-path tests where the journey itself is the thing under test.

**Accessibility assertions** could be layered in. Since the locators already use ARIA roles, adding `@axe-core/playwright` to run an automated accessibility scan on each results page would be a low-effort, high-value addition — especially relevant for a government service with legal accessibility obligations.

**Visual regression testing** would catch unintended UI changes. Playwright's built-in `toHaveScreenshot()` could be applied to the results page so that any layout regressions in the entitlement summary are caught automatically, not just logical calculation errors.

**Flakiness tracking** is something I'd formalise. The `waitForTimeout` workaround suggests there may be other timing sensitivities in the form. I'd run the suite with `--repeat-each=3` periodically in CI to surface any intermittent failures before they become production incidents.

---

### A Note on Test Scope

One broader consideration worth flagging: the current suite tests the GOV.UK calculator as a black box, which is entirely appropriate for E2E testing. But the entitlement calculations themselves (28 days, 16.5 days etc.) are business-critical figures. If the GOV.UK service ever quietly changes its rounding logic or statutory minimums, these tests would catch it — but only if someone remembered to update the expected values. I'd complement this suite with a separate layer of unit tests against the underlying calculation logic (if that code is owned by your team), so the two layers catch different categories of regression independently. BVA would be equally valuable there — ensuring unit tests systematically probe the edges of each calculation rule rather than only the comfortable mid-range values the happy-path flows currently cover.
