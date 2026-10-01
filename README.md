# pw-pom-generator

A **generator** that creates and maintains Playwright TypeScript **Page Object Model** projects for web apps.
Claude Code logs in with a generated auth script, crawls the app read-only with Playwright, then writes page objects, shared components, fixtures and smoke tests into a **target folder** you choose.
The rules it follows are in [`CLAUDE.md`](CLAUDE.md).

> **Key points**
> - This folder is the generator only. Each app gets its own **target folder**.
> - Credentials live in the **target folder's `.env`**. You fill it in yourself, never in chat.
> - The crawl is **read-only**. It never submits forms or clicks Save, Delete, Submit, Confirm or Approve.

## Quick start (already set up)

```powershell
cd <path>\pw-backbone
claude
```
Then type: `Start a session per CLAUDE.md.` and answer the questions.

## First-time setup (Windows / PowerShell)

Do this once per machine.

**1. Node.js 18+**: install the LTS version from https://nodejs.org, then check:
```powershell
node -v
```

**2. Git for Windows** (Claude Code needs it): install from https://git-scm.com/download/win, then check:
```powershell
git --version
```

**3. Claude Code**: install, then check:
```powershell
irm https://claude.ai/install.ps1 | iex
claude --version
```

**4. Log in** with your company Claude account:
```powershell
claude
```
Inside Claude Code, run `/login`, choose **Claude account with subscription**, and sign in with your work email.
Run `/status`. **Login method** should show your Team account. Exit with `/exit`.

**5. Get the generator:**
```powershell
git clone <repo-url> pw-backbone
cd pw-backbone
```

**6. Register the Playwright MCP** inside the pw-backbone folder:
```powershell
claude mcp add playwright -- cmd /c npx @playwright/mcp@latest
claude mcp list
```
`playwright` should show **Connected**.

**7. Test the browser.** Start `claude` and type:
```
Use the playwright MCP to open https://example.com and tell me the page title.
```
A browser opens and Claude replies `Example Domain`. Setup is done.

## Generate a new project

1. Create an empty target folder, e.g. `C:\Users\<you>\Documents\GitHub\my-app-tests`, and put a `.env` file in it:
```
   BASE_URL=https://your-app-url
   APP_USER=your-username
   APP_PASS=your-password
```
   You write this file yourself. Claude only reads `BASE_URL` from it and never sees the username or password.
2. Start Claude Code in the generator folder:
```powershell
   cd <path>\pw-backbone
   claude
```
3. Type: `Start a session per CLAUDE.md.`
4. Answer the questions:
   - Mode: **NEW**
   - Target folder: the absolute path from step 1
   - Page limits (default: depth 3, 30 pages)
5. If Claude asks for access to the target folder, allow it (CLI: `/add-dir <target folder path>`).
6. Claude confirms the plan, then:
   - scaffolds the Playwright project in the target folder
   - generates `tests/auth.setup.ts` and runs it. The script reads your `.env`, logs in and saves the session to `.auth/state.json`
   - crawls the app with that saved session (`.crawl/crawl.mjs`) and writes the page objects, fixtures and smoke tests
7. Check `CRAWL_REPORT.md` in the target folder.

**Tip:** for a first try on a new app, ask for small limits (depth 1, 5 pages) to check the output quality.

## Update an existing project

Same as above, but choose mode **UPDATE** and give the existing target folder.
Claude re-crawls the app and fixes broken locators. It adds new elements and pages, and marks missing ones with `// TODO: not found in last crawl`. It **never deletes** anything. Results go in the target folder's `UPDATE_REPORT.md`.

**Before an update**, commit the target project so you can review the changes with `git diff`.

## Run tests (in the target folder)

```powershell
cd <target folder>
npm run auth         # log in again and refresh .auth/state.json (when the session expires)
npm run typecheck    # tsc --noEmit
npm run test:smoke   # smoke tests, chromium only
npm test             # all tests
npm run test:ui      # Playwright UI mode
npx playwright show-report
```

## Folder structure

**Generator (this folder):**
```
pw-backbone/
  CLAUDE.md        generation and update rules for Claude Code
  README.md        this file
```

**Generated target project:**
```
<target>/
  pages/           <PageName>Page.ts: one class per page (locators, goto(), actions)
  components/      shared parts: header, sidebar, modals
  fixtures/        pages.ts: test extended with page-object fixtures
  tests/
    auth.setup.ts  logs in and saves the session to .auth/state.json
    smoke/         <page>.spec.ts: key elements are visible
  .crawl/          crawl.mjs / verify.mjs helpers; output/ is git-ignored
  .auth/           saved session (git-ignored)
  .env             BASE_URL + credentials (git-ignored, you create it)
  .env.example     placeholder keys (safe to commit)
  README.md        how to run this project's tests
  CRAWL_REPORT.md / UPDATE_REPORT.md: output of the last run
```

## Protecting manual code

Generated files can be edited by hand. Update runs leave anything you wrap in markers untouched:

```ts
// MANUAL START
async createUserWithDefaults() { /* your code */ }
// MANUAL END
```

Update runs also keep methods, assertions and tests they didn't generate. They only rename an element when its meaning has changed, and they list every rename in `UPDATE_REPORT.md`.
