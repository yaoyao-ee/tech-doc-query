---
name: tech-doc-query
description: >
  Queries official documentation sites via Playwright browser when
  WebFetch cannot access them. Use when the user references any official
  docs — API references, command syntax, usage guides, getting-started
  overviews — or provides any URL (even casually). Triggers: "official
  docs", "usage guide", "API reference", "官方文档", "官方教程",
  "怎么使用". Also use when WebFetch fails due to auth, iframe nesting,
  or domain restrictions. CRITICAL: any user message containing a URL
  MUST trigger this skill first, before any other action.
when_to_use: >
  URL triggers (highest priority — check BEFORE WebFetch):
  User sends any URL in the message, especially with casual verbs:
  "看看", "查看", "看一下", "打开", "打开看看", "帮我看看",
  "这个", "查一下", "访问", "浏览", "look at", "check out",
  "take a look", "open this", "view", "browse".

  Keyword triggers:
  "check the docs", "help page", "verify syntax", "API
  reference", "getting started", "usage guide", "summarize docs",
  "官方文档", "官方使用方式", "官方教程", "怎么使用", "帮助文档".

  Also: WebFetch returns auth/domain safety error.
allowed-tools: Bash(npx playwright *) WebFetch Read Grep
---

# Technical Documentation Query via Playwright

Query official technical documentation sites that WebFetch cannot access
due to authentication requirements, iframe nesting, or domain restrictions.

## Prerequisites

- Playwright MCP registered in Claude Code: verify with `/mcp`
- Browser channel installed: `npx playwright install chrome`
- For authenticated sites: a saved browser session (see Step 1)
- **Serial agents work; parallel agents don't.** Subagents dispatched via
  Agent() CAN use Playwright MCP tools — but only one at a time. Multiple
  parallel agents fight over the same browser tab, causing ERR_ABORTED
  and corrupt results. Run agents sequentially and wait for each to finish
  before dispatching the next. 6 sequential agent tasks were verified working.
- **Multiple tabs are supported:** Use `browser_tabs new` to open additional
  tabs within the same browser instance. Use `browser_tabs select` to switch
  between them. Each tab maintains its own URL and page state. This allows
  pre-loading or comparing multiple pages sequentially from one session.

## Pre-Flight: Diagnose Page Structure

**Before any interaction, diagnose the documentation site type.** Different
sites require different interaction patterns. Skipping this step leads to
timeout errors, "ref not found" failures, and silent mis-extraction.

1. Navigate to the target page.
2. Take a `browser_snapshot`.
3. **Check for Cloudflare or bot-blocking first.** Signs:
   - Page title is "Attention Required" or "Just a moment..."
   - Navigation timeout (>60s) with no content loaded
   - **→ Stop and report to user.** Do not retry. These sites cannot be
     accessed via Playwright.

4. Examine the snapshot for site type:

| Site Type | Snapshot Signature | Extraction Strategy |
|-----------|-------------------|-------------------|
| **iframe content** | `iframe [ref=...]` wrapping actual doc text | ALL interactions via `browser_run_code_unsafe` + frame context |
| **Standard (has `<main>`)** | `<main>` in snapshot | `browser_snapshot` or `main.textContent()` |
| **Standard (has `<article>`)** | `<article>` in snapshot | `browser_snapshot` or `article.textContent()` |
| **No semantic elements** | No `<article>`, no `<main>` | `browser_snapshot` (only reliable method) |
| **Swagger/Redoc/OpenAPI explorer** | Buttons like "POST /..." "GET /..." with collapsed sections | Fetch the underlying spec JSON/YAML instead of DOM extraction |
| **Single-page reference manual** | One giant page, 100+ headings, flat heading structure | Heading-targeted extraction (Pattern C) — do NOT extract full page |

5. **iframe ≠ content in iframe.** Seeing `iframe` in the snapshot does not
   mean the documentation content is inside it. Many sites (Oracle, SAP)
   use iframes for cookies/analytics while serving content in `<main>` or
   `<article>`. Always verify by checking the frame's URL:
   ```js
   // via browser_run_code_unsafe
   page.frames().forEach(f => console.log(f.name(), '→', f.url()));
   ```
   Content iframes will have doc-like URLs. Ignore frames with URLs
   containing `cookie`, `consent`, `analytics`, `tracking`, or empty names.

6. **Report diagnosis before proceeding.** After completing the diagnosis,
   state your findings to the user in one line before extracting content.
   Include: site framework (if identifiable), iframe status, semantic
   elements found, and which extraction method you'll use. Example:
   "MathWorks 帮助中心：无 iframe，有 `<main>`。用 Method 1 snapshot。"

7. **If the current page doesn't contain the target content, switch pages.**
   Documentation sites organize content into zones. Don't keep scanning
   the wrong zone — move to the right one based on content type:

   | Target content | Look in |
   |---------------|---------|
   | API reference / function syntax | `/reference/`, `/api/`, `/ref/` |
   | Concepts / architecture / theory | `/Concepts/`, `/About/`, `/overview/` |
   | Step-by-step tutorials | `/Tutorials/`, `/quickstart/`, `/getting-started/` |
   | How-to / troubleshooting | `/How-To-Guides/`, `/guides/`, `/faq/` |
   | Installation / setup | `/Installation/`, `/setup/`, `/downloads/` |

   Use the sidebar navigation links (visible in `browser_snapshot`) to
   jump to the correct section. If the user's URL points to a How-To
   page but they asked for architecture, don't waste time scanning it —
   go to Concepts immediately.

Known content iframes (check `reference.md` for more):
- ANSYS Help: `the_iframe`

## Standard Workflow

### Step 1: Handle Authenticated Pages

When navigation redirects to a login/sign-in page:

1. **STOP and tell the user:**
   "This page requires authentication. The login page is open. How would
   you like to proceed?"

2. **For manual login:**
   - Ask the user which login method they prefer (password, phone code, SSO).
   - Use `browser_run_code_unsafe` to fill fields and click buttons.
     `browser_type`/`browser_fill_form` refs expire quickly even on
     non-iframe pages — prefer `browser_run_code_unsafe` for all login
     interactions.
   - **CAPTCHA / human verification:** MUST pause and ask the user to
     complete it manually. Playwright cannot solve CAPTCHAs.
   - **SMS/email verification code:** Ask the user to provide the code
     when they receive it. Do not guess or retry.
   - After login succeeds (URL changes from sign-in → target page):
     notify the user.

3. **Session save (optional):**
   After login, you may ask: "Save this session for future queries?"
   If yes: `npx @playwright/mcp session save [site]-auth`
   Future sessions: `npx @playwright/mcp session load [site]-auth`

### Step 2: Load Session and Navigate

At the start of each documentation task:
```
Load the [session-name] session, then navigate to [doc starting URL].
```

Use the documentation site's table of contents (TOC) page or search
function as the entry point — never guess or manually construct URLs.

### Step 3: Navigate to the Target Page

Follow this priority order:

1. **TOC / directory links**: Use `browser_snapshot` to get the page
   structure, then navigate to the matching topic.

   **CRITICAL — MCP tools and iframes**: `browser_click`, `browser_type`,
   `browser_fill_form`, and `browser_select_option` operate on the **root
   page context only**. If the snapshot shows content inside an iframe,
   these tools will fail with "ref not found" — the ref exists in the
   iframe's DOM, not the root page. Use `browser_run_code_unsafe` with
   the correct frame context for ALL iframe interactions:

   ```js
   // CORRECT — via browser_run_code_unsafe
   const iframe = page.frames().find(f => f.name() === 'FRAME_NAME');
   await iframe.click('selector');  // or $$('a') and match by text
   await page.waitForTimeout(2000); // wait for nav
   ```

2. **Search box**: For large doc systems, search within the documentation.
   If the site is iframe-based, the search input is inside the iframe —
   locate it via the frame context, not the root page:

   ```js
   const iframe = page.frames().find(f => f.name() === 'FRAME_NAME');
   // Try multiple selector strategies
   const input = await iframe.$('input[type="text"]')
     || await iframe.$('input[type="search"]')
     || await iframe.$('textarea');
   if (input) {
     await input.fill('keyword');
     await input.press('Enter');
   }
   ```

   If standard selectors fail, fall back to keyboard interaction:
   click the search container by matching visible text, then type.

3. **Swagger / Redoc / OpenAPI explorer pages:**
   If the snapshot shows HTTP method buttons (GET/POST/PUT/DELETE) arranged
   in tagged groups (e.g., "pet", "store", "user"), this is an interactive
   API explorer. Content is rendered from an OpenAPI spec file. Instead of
   DOM extraction, fetch the underlying JSON/YAML:

   The spec URL is usually visible at the top of the page (e.g.,
   `https://petstore.swagger.io/v2/swagger.json`). Use WebFetch or
   `browser_run_code_unsafe` with `fetch()` to get structured endpoint
   and schema data from the spec file.

   `browser_snapshot` captures the collapsed endpoint list (method + path +
   summary). Expand individual endpoints by clicking them via
   `browser_run_code_unsafe` to get full parameter/response details.

**Never fabricate URLs**: Do not manually construct or guess URL
paths unless the pattern is confirmed static with no dynamic tokens
(e.g., ANSYS Help `returnurl` pattern is stable and can be used).

### Step 4: Extract Content

Choose method based on the site type diagnosed in Pre-Flight.

#### Method 1: browser_snapshot (START HERE — always try first)

```
browser_snapshot
```

**This is the only method with 100% success rate across 14 tested sites.**
Do NOT skip to Method 2 just because the page looks simple. `browser_snapshot`
returns the accessibility tree with content pre-segmented into logical blocks
— it naturally filters out CSS, `<style>` blocks, sidebars, and navigation
noise that DOM extraction methods can include.

**Limitation:** May produce very large output for single-page manuals
(1.3MB+). If output >100KB, switch to heading-targeted extraction (Method 3).

#### Method 2: DOM container extraction (fallback — only if snapshot too large)

Use this ONLY when browser_snapshot output exceeds 100KB and you need a
smaller result. Even then, consider heading-targeted extraction (Method 3)
first for large pages. Note: `main.textContent()` can include unexpected
noise — MathWorks pages embed 5KB+ of `<style>` CSS inside `<main>`.

```js
// via browser_run_code_unsafe
const content = await page.evaluate(() => {
  const el = document.querySelector('article')
    || document.querySelector('main')
    || document.querySelector('#content-area, #content, .mdx-content, [role="main"]');
  if (!el) return document.body.innerText.substring(0, 5000);
  // Sphinx/RTD: exclude injected chrome
  const clone = el.cloneNode(true);
  clone.querySelectorAll('nav, .downloads, .versions, [class*="injected"]')
    .forEach(n => n.remove());
  return clone.textContent.trim().substring(0, 15000);
});
```

**body.textContent() WARNING:** Do NOT use unscoped body extraction.
Tested body sizes: GitBook 296KB, GitHub 515KB, Auth0 7.7MB, GNU 524KB.

#### Method 3: Heading-targeted extraction (for large pages)

For single-page manuals (GNU/texinfo) or pages with 100+ headings:

```js
const sections = await page.$$eval('h2, h3, h4', (headings) => {
  return headings.map(h => {
    const level = parseInt(h.tagName[1]); // 2, 3, or 4
    let content = '';
    let next = h.nextElementSibling;
    while (next) {
      const tag = next.tagName;
      if (/^H[2-4]$/.test(tag)) {
        const nextLevel = parseInt(tag[1]);
        if (nextLevel <= level) break; // stop at same-or-higher heading
        // lower-level sub-headings pass through (included in this section)
      }
      content += next.textContent + '\n';
      next = next.nextElementSibling;
    }
    return { heading: h.textContent.trim(), tag: h.tagName, level, content: content.trim() };
  });
});
// Filter: sections.filter(s => s.heading.includes('keyword'))
```

The level-aware boundary means H3 "Shell Functions" includes H4
sub-sections under it, only stopping at the next H3 or H2.

#### Method 4: Swagger/OpenAPI spec fetch (for API explorers)

Fetch the raw `swagger.json` / `openapi.yaml` URL shown on the page.
Use WebFetch or `browser_run_code_unsafe` with `fetch()`.
Parse `spec.paths` for endpoints and `spec.definitions` for schemas.

#### Method 5: Substring pagination (for very long pages)

When `browser_run_code_unsafe` output exceeds 15,000 characters, split
at section boundaries (heading elements) rather than arbitrary character
offsets. Arbitrary cuts can break tables and code blocks in half:

```js
// via browser_run_code_unsafe — returns chunks split at H2/H3 boundaries
const chunks = await page.evaluate(() => {
  const el = document.querySelector('article') || document.querySelector('main')
    || document.querySelector('#content-area, #content');
  if (!el) return [document.body.innerText.substring(0, 15000)];
  const result = [];
  let current = '';
  const MAX_LEN = 15000;
  for (const child of el.children) {
    const text = child.textContent;
    if (current.length + text.length > MAX_LEN && current.length > 0) {
      result.push(current);
      current = text;
    } else {
      current += (current ? '\n' : '') + text;
    }
  }
  if (current) result.push(current);
  return result;
});
// chunks[0], chunks[1], etc.
```

### Step 5: Batch Processing (Multiple Pages)

When the user needs to look up multiple topics across different pages:

**Dispatch agents sequentially, not in parallel.** Subagents CAN use
Playwright MCP, but parallel agents fight over one browser tab.
Run one agent at a time, wait for completion, then dispatch the next.
6 sequential agent tasks were verified working.

```
# Option A: Sequential with tabs ← recommended for 2-4 pages
1. browser_navigate to Page A → snapshot → extract
2. browser_tabs new → browser_navigate to Page B in tab 1 → snapshot → extract
3. browser_tabs select 0 → back to Page A content
4. (Optional) Dispatch summary agent with all extracted text

# Option B: Simple sequential ← for single pages or simple flows
1. Navigate → snapshot → extract → repeat
```

### Step 6: Output Format

For each documentation page, return a structured summary:

- **Name**: Command, API, or function name
- **Syntax**: Exact syntax with all parameters
- **Parameters**: Table of parameter names, types, defaults, descriptions
- **Notes**: Important caveats, version restrictions, deprecation notices
- **Source**: Full URL of the page consulted

If the user requests verbatim text, output the complete extracted
content instead.

## Critical Rules

- **Extraction flow: Method 1 first, always.** Start every extraction with
  `browser_snapshot`. Do NOT skip to Method 2 (DOM extraction) for pages
  that "look simple" — even clean-looking server-rendered pages can have
  unexpected noise (MathWorks: 5KB of `<style>` CSS inside `<main>`).

- **Never reach for browser_click first.** On ANY page — iframe or not —
  refs expire fast. `browser_click` failed twice in real usage on
  server-rendered pages (DeepSeek login, MathWorks search). Default to
  `browser_run_code_unsafe` for all interactions. Only use browser_click
  when you have a fresh snapshot and know the ref is in root page context.

- **MCP interaction tools (browser_click, browser_type, browser_fill_form,
  browser_select_option) operate on the root page only.** They cannot reach
  elements inside an iframe. For ALL iframe interactions, use
  `browser_run_code_unsafe` with `page.frames().find(f => f.name() === 'NAME')`.

- **Stop at login pages.** If navigation redirects to a sign-in page, do
  not attempt to bypass it. Ask the user how to proceed. For CAPTCHA or
  SMS verification codes, pause and wait for the user.

- **Stop at Cloudflare / bot-blocking.** If the page title is "Attention
  Required" or navigation times out with no content, report to the user.
  Do not retry.

- **iframe ≠ content in iframe.** Check the frame's URL before assuming
  it contains the documentation. Many sites (Oracle, SAP) use iframes
  for cookies/analytics only.

- **Refresh snapshot before every action.** Refs expire after any
  navigation or state change.

- **Never use WebFetch for authenticated domains.** Use Playwright
  browser navigation instead.

- **Scope text extraction.** Avoid `document.body.innerText` and broad
  `body.textContent()`. Tested body sizes: GitBook 296KB, GitHub 515KB,
  Auth0 7.7MB, GNU 524KB. Use browser_snapshot, targeted container
  extraction, or heading-targeted extraction.

- **Swagger/Redoc is not prose documentation.** Fetch the underlying
  OpenAPI spec JSON/YAML instead of trying to extract rendered HTML.

- **Single-page manuals need section targeting.** GNU/texinfo-style pages
  pack hundreds of headings into one HTML file. Always use
  heading-targeted extraction — never extract the full page.

- **Ref-based tools expire fast.** `browser_type`, `browser_fill_form`,
  and `browser_click` can fail with "ref not found" even on simple
  server-rendered pages if there's any delay. For form-filling and
  multi-click workflows, use `browser_run_code_unsafe` instead.

## Post-Query

Keep the browser session open within the conversation unless the user
explicitly asks to close it. Subsequent queries can reuse the same
session without re-authentication.

## Before Each Query

1. **Check `reference.md`** for the site's iframe name, known quirks, and
   verified extraction pattern.
2. **If the site is not documented:** run Pre-Flight diagnosis, then add
   findings to `reference.md` using the Case Study Template.
