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
- Never write credentials into any generated file; code reads them from process.env only
- NEVER ask for credentials in chat

# Set up the target project (NEW mode only)
- Scaffold a Playwright TypeScript project in the target folder, chromium only, no example tests:
  @playwright/test, typescript, @types/node, dotenv; then npx playwright install chromium
- playwright.config.ts: load dotenv; use.baseURL = process.env.BASE_URL; projects:
  - setup: testMatch /auth\.setup\.ts/
  - chromium: dependencies ['setup'], use.storageState '.auth/state.json'
- .gitignore must include: .env, .auth/, .crawl/output/, node_modules/, test-results/,
  playwright-report/, blob-report/, playwright/.cache/
- Create .env.example with placeholders only: BASE_URL, APP_USER, APP_PASS
- Run npx tsc --noEmit to confirm the setup works

# Login (auth script + session file)
Claude never types credentials into a browser itself and never sees their values.
1. Open BASE_URL logged out (Playwright MCP or a script) and read the login form. Look only; do not type
2. Generate tests/auth.setup.ts: fill APP_USER / APP_PASS from process.env, submit, wait for a
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

# Output structure
- pages/<PageName>Page.ts: one class per page, constructor(page: Page), readonly locators,
  a goto() method, basic action methods (e.g. fillUsername, clickSave)
- components/<Name>.ts: shared parts (header, sidebar, modals), used by pages
- fixtures/pages.ts: extends Playwright test with all page objects as fixtures
- tests/auth.setup.ts: logs in and saves .auth/state.json (see Login)
- tests/smoke/<page>.spec.ts: open the page, assert its key elements are visible
- .crawl/crawl.mjs, .crawl/verify.mjs: crawl helpers (output in .crawl/output/, git-ignored)
- CRAWL_REPORT.md: list of pages found, elements with no stable locator, pages skipped and why
- README.md: what the project covers, setup (.env), how to run tests, folder structure,
  the MANUAL START/END rule. Short and scannable.
- package.json scripts: "test", "test:smoke", "test:ui", "typecheck", "auth" (runs the setup project)

# Naming
- Meaningful names based on visible text or purpose: loginButton, searchInput, usersTable
- Never generic names like button1, input3
- Page class names from the page title or route: /users/list -> UsersListPage

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
