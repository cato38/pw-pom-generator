# What this folder is
This folder is the GENERATOR, not a test project.
Never write generated project files (pages/, tests/, fixtures/, components/, reports, node_modules/) into this folder.
Every generated project lives in its own TARGET folder that the user gives.
All paths below are relative to the TARGET folder.

# Start of every session (always do this first)
Ask the user, then wait for answers:
1. Mode: NEW project or UPDATE existing project?
2. Target folder: absolute path (e.g. C:\Users\cagat\Documents\GitHub\my-app-tests)
3. Page limits (default: max depth 3, max 30 pages)
Then check the target:
- If you cannot access the target folder, request access to it
  (desktop app: directory access request; CLI: tell the user to run /add-dir <target path>)
- .env must exist and have non-empty BASE_URL, APP_USER, APP_PASS (check per "Handling .env");
  otherwise stop and ask the user to fill it
- NEW: the folder must contain only .env (a .env.example / .gitignore is fine); otherwise stop and ask
- UPDATE: the folder must contain pages/; otherwise stop and ask
- Repeat the plan back in 3 lines (mode + target, BASE_URL, page limits) and wait for OK before starting

# Handling .env (secrets)
- The user creates and owns .env. Never create, overwrite, or edit it
- Read only the BASE_URL line (e.g. grep "^BASE_URL=" .env)
- Never read, print, or echo APP_USER / APP_PASS. Check they are set without showing them
  (e.g. grep -c "^APP_PASS=." .env)
- Never write credentials into any generated file; code reads them from process.env only, through config/env.ts
- NEVER ask for credentials in chat

# Set up the target project (NEW mode only)
- Scaffold a Playwright TypeScript project in the target folder, chromium only, no example tests:
  @playwright/test, typescript, @types/node, dotenv; then npx playwright install chromium
- Create the files and folders listed in "Project structure" below (empty folders get no placeholder files)
- Create .env.example with placeholders only: BASE_URL, APP_USER, APP_PASS
- Run npx tsc --noEmit to confirm the setup works

# Login (auth script + session file)
Claude never types credentials into a browser itself and never sees their values.
1. Open BASE_URL logged out (Playwright MCP or a script) and read the login form. Look only; do not type
2. Generate tests/setup/auth.setup.ts: fill env.user (APP_USER / APP_PASS via config/env.ts), submit, wait for a
   logged-in signal (URL leaves the login route or a logged-in-only element is visible), then save
   the session to .auth/state.json. Never log the credential values
3. Run it: npx playwright test --project=setup
   If it fails, show the error and ask the user to check .env. Never read the password to debug
4. The crawl and the tests reuse .auth/state.json. If the session has expired (redirect to login), re-run step 3

# Crawl
The crawl runs as a script in the target folder so it can reuse the saved session.
The Playwright MCP browser is not logged in; use it only for logged-out pages (e.g. the login page)
or to look at a page by hand.
- .crawl/crawl.mjs: uses chromium from @playwright/test with storageState .auth/state.json, starts at BASE_URL
- For each page found it saves .crawl/output/<route>.json (git-ignored): URL, title, the page's
  aria snapshot, elements with data-testid, links, nav/menu/sidebar items
- Run it with node, read the output, and write the page objects from it
- .crawl/verify.mjs: given candidate locators per page, opens each page with the saved session and
  prints the match count of each locator. Use it for the "exactly 1 element" check
- UPDATE: reuse the existing .crawl/ scripts; adjust them if the app changed

# Crawl rules
- Same-origin pages only; use the page limits from the session start
- READ-ONLY: never submit forms, never type into fields, never click Save / Delete / Submit /
  Confirm / Approve / Send or anything similar
- Never click Logout / Sign out (it ends the saved session)
- Discover pages by following links AND clicking nav/menu/sidebar items
- SPA: treat each route change as a separate page
- Login page: crawl it logged out; its smoke test uses an empty storageState
  (test.use({ storageState: { cookies: [], origins: [] } }))

# Locator rules
- Priority: getByTestId > getByRole(name) > getByLabel > getByPlaceholder > CSS
- Each locator must match exactly 1 element; verify with .crawl/verify.mjs before writing it
- CSS is the last resort; mark it with a // TODO: unstable locator comment
- No XPath, no nth-child, no auto-generated class names

# Project structure
Goal: a small, predictable backbone. One place for each kind of thing, one way to import it.
```
<target>/
├── config/
│   └── env.ts                    the ONLY file that reads process.env (typed, fails fast)
├── pages/
│   ├── base.page.ts              BasePage: page, url, goto(), expectLoaded(), shared helpers
│   └── <name>.page.ts            one class per page, extends BasePage
├── components/
│   └── <name>.component.ts       shared UI parts (header, sidebar, modal, table)
├── fixtures/
│   └── index.ts                  the ONLY `test` / `expect` that specs import
├── tests/
│   ├── setup/auth.setup.ts       logs in, saves .auth/state.json
│   ├── smoke/<name>.spec.ts      generated: page opens, key elements visible
│   └── e2e/<feature>/*.spec.ts   manual user-flow tests (generator never writes here)
├── data/                         static test data (no secrets), e.g. users.data.ts
├── utils/                        pure helpers with no page knowledge (dates, downloads, random)
├── .crawl/                       crawl.mjs, verify.mjs (output/ is git-ignored)
├── playwright.config.ts
├── tsconfig.json
├── package.json
├── .env / .env.example
├── .gitignore
├── README.md
└── CRAWL_REPORT.md / UPDATE_REPORT.md
```

## Dependency direction (never import upwards)
tests -> fixtures -> pages -> components -> base.page / utils / config
- Specs never import from @playwright/test, pages, or config directly; everything comes from fixtures/index.ts
- Pages never import specs or fixtures; components never import pages
- Only config/env.ts touches process.env (except playwright.config.ts loading dotenv)

## config/env.ts
- Exports one `env` object: baseURL, user { username, password } from BASE_URL / APP_USER / APP_PASS
- A required() helper throws a clear error naming the missing variable (never its value)
- No hard-coded URLs, no per-environment files. Another environment = another .env file
  (optional ENV_FILE variable picks it in playwright.config.ts)

## pages/
- File: kebab-case `<name>.page.ts`; class: PascalCase + Page (`users-list.page.ts` -> UsersListPage)
- Every page extends BasePage and sets `readonly url` (relative to baseURL); BasePage.goto() uses it
- Class layout, always in this order, separated by one-line section comments:
  1. `// Locators` readonly Locator fields
  2. `// Components` readonly component instances (header, sidebar) if the page has them
  3. constructor(page: Page): super(page), then assigns locators in the same order as declared
  4. `// Actions` small methods named for intent: search(text), openUser(name), clickSave()
  5. `// Assertions` only expectLoaded() (checks the page's key element); other expects stay in specs
- No test logic, no hard waits (waitForTimeout), no credentials, no try/catch around locators
- Methods return Promise<void>, or a value the test needs; navigation methods may return the next page object

## components/
- File: `<name>.component.ts`; class: PascalCase + Component (HeaderComponent)
- constructor(page: Page) or constructor(root: Locator) for repeated parts; locators are scoped to the root
- Create a component only when the part appears on 2+ pages; otherwise keep it in the page

## fixtures/index.ts
- One `base.extend<Fixtures>()` that registers every page object as a camelCase fixture (usersListPage)
- Re-exports `expect`. New page = one new line in the Fixtures type + one fixture entry
- Specs: `import { test, expect } from '@fixtures';`

## tests/
- setup/auth.setup.ts: see Login. Uses env from config, nothing else
- smoke/<name>.spec.ts: one file per page, one `test.describe('<PageName>')`, tagged @smoke:
  goto(), expectLoaded(), then toBeVisible() on its key elements. Nothing that changes data
- e2e/: owned by the user. Tag tests @regression (plus a feature tag like @users)
- Test titles describe behaviour: 'shows the users table', not 'test 1'

## Config files
- tsconfig.json: strict, noEmit, paths aliases `@pages/*`, `@components/*`, `@fixtures`, `@config/*`,
  `@data/*`, `@utils/*` (no ../../ imports between top-level folders)
- playwright.config.ts: load dotenv first; testDir ./tests; fullyParallel; forbidOnly on CI;
  retries 2 on CI / 0 local; reporter list + html (open: 'never'); use.baseURL = env.baseURL,
  trace 'on-first-retry', screenshot 'only-on-failure', video 'retain-on-failure'; projects:
  - setup: testMatch /auth\.setup\.ts/
  - chromium: devices['Desktop Chrome'], dependencies ['setup'], use.storageState '.auth/state.json'
- .gitignore: .env, .env.*, !.env.example, .auth/, .crawl/output/, node_modules/, test-results/,
  playwright-report/, blob-report/, playwright/.cache/
- package.json scripts: "test", "test:smoke" (--grep @smoke), "test:regression" (--grep @regression),
  "test:ui" (--ui), "typecheck" (tsc --noEmit), "auth" (--project=setup), "report" (show-report)

## Docs
- README.md: what the project covers, setup (.env), how to run tests, folder structure,
  the MANUAL START/END rule. Short and scannable
- CRAWL_REPORT.md: pages found, elements with no stable locator, pages skipped and why

## Not in the backbone (add only when the user asks)
API layer (api/clients, api/services, api/types), other browsers, custom reporters/webhooks,
Docker, CI pipeline files, multiple environment files

# Naming
- Meaningful names based on visible text or purpose: loginButton, searchInput, usersTable
- Never generic names like button1, input3
- Suffix locators with their role: Button, Link, Input, Select, Checkbox, Table, Heading, Dialog
- Page names from the page title or route: /users/list -> users-list.page.ts / UsersListPage / usersListPage
- Files kebab-case, classes PascalCase, fixtures/methods/locators camelCase

# When done
- Run npx tsc --noEmit and fix any type errors
- Run the smoke tests with --project=chromium and report results

# Update mode (UPDATE existing project)
## Steps
1. Refresh the session (npx playwright test --project=setup), then re-crawl with .crawl/crawl.mjs
   using the same crawl rules
2. For every existing page object:
   - Check each locator still matches exactly 1 element (.crawl/verify.mjs)
   - Broken locator -> find a new one using the locator priority; replace it
   - New element on the page -> add locator (+ basic action method if interactive)
   - Element no longer on the page -> do NOT delete; mark with
     // TODO: not found in last crawl (YYYY-MM-DD) - verify and remove
3. New page found -> create its page object, fixture entry, and smoke test
4. Page no longer reachable -> do NOT delete files; list it in the report
5. Run npx tsc --noEmit and the smoke tests (--project=chromium); fix what the update broke
6. Update README.md if the structure changed

## Existing structure
- If the project does not follow "Project structure" (e.g. older PascalCase files), keep its
  conventions for new files and list the differences in the report. Restructure only if the user asks

## Protect manual work
- Never delete or rewrite methods, assertions, or tests that were not generated by you
- Code between // MANUAL START and // MANUAL END is never touched
- Keep existing names; rename only if the element's meaning changed, and list renames in the report

## Report
Write UPDATE_REPORT.md in the target folder:
- Locators changed (old -> new, with page and reason)
- Elements added / marked as not found
- Pages added / unreachable
- Renames
- Test results after update
