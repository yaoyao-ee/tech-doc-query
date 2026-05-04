# Technical Documentation Query — Reference

## General Diagnosis Methodology

When encountering a new documentation site, diagnose before interacting:

1. **Navigate** to the target URL.
2. **Take a `browser_snapshot`** and examine the structure:
   - Is there an `iframe` wrapping content? → **iframe-based site**
   - No iframe, but content loads dynamically? → **SPA / JS-rendered**
   - No iframe, plain HTML with standard links? → **Server-rendered**
3. **For iframe sites**, immediately discover the frame name:
   ```js
   // via browser_run_code_unsafe
   page.frames().forEach(f => console.log(f.name(), f.url()));
   ```
4. **Test a single navigation** to confirm the interaction pattern works.
5. **Document findings** below as a new case study.

## Common Site Patterns and Interaction Rules

| Site Type | Identify By | Click/Target Links | Type/Form Fill | Extract Content |
|-----------|------------|-------------------|---------------|-----------------|
| **iframe-based** | `iframe` in snapshot | `browser_run_code_unsafe` + frame context ONLY | Same — via frame context | `browser_snapshot` or frame-targeted JS |
| **SPA / JS-rendered** | No iframe, dynamic updates | `browser_click` works | `browser_type` works | `browser_snapshot` |
| **Server-rendered** | No iframe, static HTML | `browser_click` works | `browser_type` works | `browser_snapshot` |

**Core rule**: `browser_click`, `browser_type`, `browser_fill_form`, and
`browser_select_option` operate on the **root page context only**. They
cannot reach elements inside an iframe. For iframe content, all
interactions go through `browser_run_code_unsafe` with the frame object
obtained via `page.frames().find(f => f.name() === 'FRAME_NAME')`.

## Verified Extraction Patterns

### Pattern A: browser_snapshot (preferred for all site types)

```
browser_snapshot
```

Returns the accessibility tree. Content is pre-segmented into logical
blocks — sidebars and navigation are naturally separated. Works for all
site types including iframe-based. May produce very large output for
content-heavy pages.

### Pattern B: Targeted iframe extraction (fallback for large pages)

```js
// via browser_run_code_unsafe
const iframe = page.frames().find(f => f.name() === 'FRAME_NAME');
const content = await iframe.evaluate(() => {
  const article = document.querySelector('article');
  if (!article) return document.body.innerText;
  // Collect article children, skipping nav elements
  const sections = article.querySelectorAll(':scope > *');
  let text = '';
  for (const s of sections) {
    if (s.tagName === 'NAV') continue;       // skip TOC navigation
    if (s.matches('nav, .sidebar, .toc')) continue; // skip by class
    text += s.textContent.trim() + '\n\n';
  }
  return text;
});
```

### Pattern C: Heading-based section extraction (for specific sections)

```js
// via browser_run_code_unsafe
const iframe = page.frames().find(f => f.name() === 'FRAME_NAME');
const sections = await iframe.$$eval('h2, h3, h4', (headings) => {
  return headings.map(h => {
    let content = '';
    let next = h.nextElementSibling;
    while (next && !next.matches('h2, h3, h4')) {
      content += next.textContent + '\n';
      next = next.nextElementSibling;
    }
    return { heading: h.textContent.trim(), content: content.trim() };
  });
});
// Filter by heading text match
const target = sections.find(s => s.heading.includes('keyword'));
```

### Pattern D: Substring-based pagination (for very long pages)

When content exceeds 15,000 characters:
```js
// First call: content.substring(0, 15000)
// Second call: content.substring(15000)
```
Combine the results for the full page content.

---

## Case Study: ANSYS Help (ansyshelp.ansys.com)

### Site Characteristics

- Public access without login for base documentation
- Customer login for additional content (tutorials, downloads)
- All documentation rendered inside `<iframe name="the_iframe">`
- Page content loaded via `returnurl` query parameter
- URL pattern: `.../public/account/secured?returnurl=////Views/Secured/corp/v242/en/...`
- Sidebar TOC embedded in the same `<article>` as main content
- Search box inside iframe, uses non-standard markup

### Entry Pattern

1. Navigate to the main product page with the `returnurl` parameter:
   ```
   https://ansyshelp.ansys.com/public/account/secured?returnurl=////Views/Secured/prod_page.html?pn=Mechanical%20APDL&prodver=24.2&lang=en
   ```
2. `browser_snapshot` to get the TOC link list.
3. Navigate to specific pages by constructing `returnurl` with the
   relative path from the TOC (URL pattern is stable):
   ```
   https://ansyshelp.ansys.com/public/account/secured?returnurl=////Views/Secured/corp/v242/en/ans_cou/Hlp_G_COU3_thermele.html
   ```

### Iframe Operation Pattern

```js
// CORRECT
const iframe = page.frames().find(f => f.name() === 'the_iframe');
await iframe.$('selector');
await iframe.click('selector');       // works inside browser_run_code_unsafe
await iframe.evaluate(() => { ... }); // for content extraction

// WRONG — do not do this
await page.click('selector');         // operates on root, misses iframe
```

### Failed Methods (Corrected Diagnoses)

| Method | Failure Reason | Correct Alternative |
| :--- | :--- | :--- |
| `browser_click` with iframe ref | MCP tool operates on root page context; iframe refs don't exist in root DOM | Use `browser_run_code_unsafe` with `page.frames().find(f => f.name() === 'the_iframe')` |
| `browser_type` with iframe ref | Same — ref not found in root page context | Same as above |
| `browser_fill_form` with iframe refs | Same — all MCP interaction tools are root-page-only | Use `browser_run_code_unsafe` for all iframe form interactions |
| Direct URL to `Hlp_C_CMD.html` | 404 — requires `returnurl` mechanism | Use the full secured URL pattern |
| `WebFetch` on `ansyshelp.ansys.com` | Domain safety verification blocked | Use Playwright browser navigation |
| `article.textContent()` (unscoped) | Returns sidebar TOC mixed with content (~93KB) | Exclude `<nav>` elements or use heading-targeted extraction |

### General Failed Methods (All Sites)

| Method | Failure Reason | Correct Alternative |
| :--- | :--- | :--- |
| `body.textContent()` | Tested: 296KB–7.7MB noise across sites | Use `main.textContent()`, `article.textContent()`, or `browser_snapshot` |
| `article.textContent()` (blindly) | Only 2/12 sites use `<article>` wrapper | Diagnose first: check for article, then main, then #content-area |
| `browser_click` with iframe ref | MCP tool operates on root page context | Use `browser_run_code_unsafe` with frame context |
| `browser_type`/`browser_fill_form` (even on non-iframe) | Refs expire quickly; "ref not found" error | Use `browser_run_code_unsafe` for any multi-step form interaction |
| Full-page extraction on single-file manuals | 524KB body, 154 headings | Use heading-targeted extraction, navigate by TOC anchor |
| DOM extraction on Swagger/Redoc | Content rendered dynamically from spec JSON | Fetch the underlying OpenAPI spec file |
| `main.textContent()` with CSS noise | MathWorks pages embed 5KB+ `<style>` blocks inside `<main>` | Use `browser_snapshot` instead, or strip `<style>` elements from clone |

### Oracle Help Center (docs.oracle.com)
- **Type:** server-rendered
- **iframe:** 3 (cookies/analytics only — NOT content containers)
- **Semantic:** `<article>` ✅, `<main>` ✅
- **Extraction:** `article.textContent()` (39.8KB clean) or `browser_snapshot`
- **Quirks:** Has iframes but content lives in article. Do not use frame context.

### SAP Help Portal (help.sap.com)
- **Type:** server-rendered
- **iframe:** 2 (non-content)
- **Semantic:** `<main>` ✅, no `<article>`
- **Extraction:** `main.textContent()` (2.4KB clean) or `browser_snapshot`
- **Quirks:** Similar to Oracle — iframes exist but content is in main.

### Sphinx / Read the Docs (*.readthedocs.io, docs.ros.org)
- **Type:** server-rendered (Sphinx + sphinx_rtd_theme)
- **iframe:** none
- **Semantic:** `<main>` ✅, no `<article>`
- **Extraction:** `browser_snapshot` preferred. `main.textContent()` needs filtering — RTD injects downloads/versions chrome at the top. Skip `.downloads`, `.versions` classes.
- **Quirks:**
  - RTD floating version banner is outside `<main>`. No `<article>`.
  - **Search is JS-dynamic:** `search.html?q=keyword` returns empty — the search
    page loads results dynamically via JS. Use the search textbox in the page
    (via `browser_run_code_unsafe`) instead of constructing search URLs.
  - **Relative URLs in sub-pages:** Links from sub-directory pages (e.g.,
    `Concepts/Basic.html`) use relative paths like `Basic/About-Nodes.html`.
    When navigating from such a page, construct the full URL from the current
    directory path, not the domain root. Check `page.url()` to see the current
    base path before constructing the next URL.

### GitBook (docs.gitbook.com)
- **Type:** SPA (Next.js)
- **iframe:** none
- **Semantic:** `<main>` ✅, no `<article>`
- **Extraction:** `main.textContent()` (817 chars clean) or `browser_snapshot`
- **Quirks:** `body.textContent()` = 296KB (all pages preloaded). Never use body. SPA navigation: browser_click works, wait ~2s for render.

### GitHub Docs (docs.github.com)
- **Type:** server-rendered + JS enhancement
- **iframe:** none
- **Semantic:** `<main>` ✅, no `<article>`
- **Extraction:** `browser_snapshot` preferred. `main.textContent()` is clean but 93KB on detail pages. Use heading-targeted extraction for specific endpoints.
- **Quirks:** body = 515KB. Sidebar nav is outside main.

### Twilio API Docs (www.twilio.com/docs)
- **Type:** server-rendered
- **iframe:** none
- **Semantic:** `<article>` ✅, `<main>` ✅
- **Extraction:** `article.textContent()` (1.1KB clean) or `browser_snapshot`
- **Quirks:** Right "Page Tools" column collapsed by default.

### Microsoft Learn (learn.microsoft.com)
- **Type:** server-rendered
- **iframe:** none
- **Semantic:** `<main>` ✅, `<article>` only in card components
- **Extraction:** `main.textContent()` (5.3KB) or `browser_snapshot`
- **Quirks:** Cookie consent banner on first visit — click Accept/Reject first.

### MathWorks Help Center (ww2.mathworks.cn/help)
- **Type:** server-rendered
- **iframe:** none
- **Semantic:** `<main>` ✅, no `<article>`
- **Extraction:** `browser_snapshot` STRONGLY preferred. `main.textContent()` includes 5KB+ of `<style>` CSS blocks at the top — never use without first checking for `<style>` noise.
- **Quirks:** MATLAB doc pages embed color-theme CSS inside `<main>`. URL pattern is stable: `https://ww2.mathworks.cn/help/<product>/ref/<function>.html`. Direct navigation is safe.

### VitePress (vitepress.dev, vuejs.org, vitejs.dev)
- **Type:** SSG (static HTML first, SPA after hydration)
- **iframe:** none
- **Semantic:** `<main>` ✅, no `<article>`
- **Extraction:** `main.textContent()` (4.6KB extremely clean). Best noise ratio: body only 5.4KB.
- **Quirks:** Pages include LLM hint: "Are you an LLM? You can read better optimized documentation at [url].md" — raw markdown source at same URL with `.md` extension.

### Auth0 Docs (auth0.com/docs)
- **Type:** MDX/Docusaurus platform
- **iframe:** none
- **Semantic:** NO `<article>`, NO `<main>`. Uses `#content-area > #content.mdx-content`
- **Extraction:** `browser_snapshot` is the ONLY reliable method. DOM fallback: `#content-area`. body = 7.7MB.
- **Quirks:** Worst body noise of all tested sites. Skip link targets `#content-area`.

### Swagger UI / Redoc (Interactive API Explorers)
- **Type:** Pure JS-rendered API browser
- **iframe:** none
- **Semantic:** NO `<article>`, NO `<main>`
- **Extraction:** Fetch the underlying OpenAPI spec JSON/YAML. The spec URL is displayed at top of page. `browser_snapshot` captures collapsed endpoint list.
- **Quirks:** Default view shows method + path + summary. Expand via `browser_run_code_unsafe` click for details.

### GNU / Texinfo Single-Page Manuals (gnu.org)
- **Type:** Pure static HTML, single file = entire manual
- **iframe:** none
- **Semantic:** NO `<article>`, NO `<main>`, flat heading structure
- **Extraction:** MUST use Method 3 (heading-targeted) with level-aware boundary.
  Tested: 524KB body, 154 headings. Use `#SEC_Contents` anchor to locate TOC,
  then navigate to specific section by heading text match.
- **Quirks:** Navigation via [Next][Previous][Up][Contents][Index] links.
  Never extract full page. The heading code in Method 3 uses level-aware
  stopping (H3 includes H4 sub-sections; stops at next H3 or H2).

### Cloudflare-Blocked Sites
- **Examples:** Stripe Docs (docs.stripe.com), OpenAI Platform (platform.openai.com)
- **Symptoms:** Page title "Attention Required! | Cloudflare", navigation timeout >60s
- **Action:** Report to user immediately. Do NOT retry. Headless browsers are blocked.

---

## Case Study Template

When documenting a new site, use this structure:

```markdown
### Site Characteristics
- Auth requirement: [none / login / SSO]
- iframe: [none / name=X]
- Content loading: [server-rendered / SPA / iframe]
- URL pattern: [stable / dynamic / requires tokens]
- Known quirks: [DOM oddities, search behavior, etc.]

### Entry Pattern
[How to reach the documentation TOC or search page]

### Extraction Notes
[Which extraction pattern works best, what to avoid]
```

---

## Session Management

### Saving a Session

After manual login, save the browser state:

```
Save the current browser session as [name].
```

Underlying command: `npx @playwright/mcp session save [name]`

### Loading a Session

At conversation start:

```
Load the [name] session.
```

Underlying command: `npx @playwright/mcp session load [name]`

### Session Expiry

If a saved session stops working (e.g., redirects to login):

1. Re-navigate to the login page.
2. Ask the user to manually log in again.
3. Re-save the session under the same name.

## Pre-Approved Tool Usage

The `allowed-tools` frontmatter grants automatic approval for:

- `Bash(npx playwright *)`: All Playwright CLI operations (install,
  session save/load, browser launch)
- `WebFetch`: Only use for domains confirmed safe and unauthenticated
- `Read`: Reading extracted content saved to files
- `Grep`: Searching within extracted content

Other Playwright browser tools (navigate, snapshot, click, type,
evaluate) are inherent to the MCP server and do not require additional
permissions when the skill is active.

## Troubleshooting

| Symptom | Cause | Solution |
| :--- | :--- | :--- |
| Click ref "not found" on iframe site | `browser_click` operates on root page, not iframe | Use `browser_run_code_unsafe` with frame context |
| Click ref "not found" on non-iframe site | Snapshot stale after page change | Take a fresh `browser_snapshot` before using refs |
| `browser_type` ref "not found" | iframe content unreachable from root | Use `browser_run_code_unsafe` + frame context |
| iframe interaction timeout | Operating from wrong frame context | Verify frame name with `page.frames().forEach(...)` |
| Page returns 404 | Guessed URL or missing dynamic token | Go back to TOC or search entry point; check URL pattern |
| Session loads but redirects to login | Session expired | Re-login and re-save the session |
| `WebFetch` returns safety error | Domain not in allowlist | Switch to Playwright immediately |
| Extracted text is all navigation links | Sidebar included in extraction scope | Skip `<nav>` elements; use `browser_snapshot` |
| Snapshot output too large (>90KB) | Very content-heavy page | Use heading-targeted extraction (Pattern C) or substring pagination (Pattern D) |
| Search input not found with `input[type="text"]` | Non-standard search markup (e.g., ANSYS Help) | Try `input[type="search"]`, `textarea`, or keyboard-fallback via text matching |
| "Browser missing Chromium" | Playwright MCP cannot find installed browser | Run `npx playwright install chrome` in admin terminal, or set `--executable-path` in MCP config |
| Page redirects to /sign_in or /login | Authentication required | Follow Step 1: ask user, handle CAPTCHA/verification code, optionally save session |
| Page title "Attention Required" or "Just a moment..." | Cloudflare bot detection | Report to user. Do not retry. |
| Navigation timeout >60s with no content | Likely Cloudflare or heavy JS | Report to user. Try a different site in same category. |
| `browser_type` ref not found on non-iframe page | Refs expire quickly even on simple pages | Use `browser_run_code_unsafe` for form filling |
| Swagger/Redoc page with collapsed endpoints | Not a prose doc — API explorer | Fetch the OpenAPI spec JSON URL shown on the page |
| Snapshot >1MB | Single-page reference manual (GNU/texinfo) | Use heading-targeted extraction or navigate by TOC anchor |
| Cookie consent banner blocking content | GDPR compliance popup | Click "Accept" or "Reject" button in snapshot before extracting |

## Quick Command Reference for Users

| Request | What to Say |
| :--- | :--- |
| Look up one command | Look up the `*DO` command in the ANSYS Help |
| Check official syntax | Verify the syntax for `git rebase` from the official docs |
| Batch verify multiple items | Check these 5 API endpoints against the reference docs |
| Search for a topic | Search the Python docs for "asyncio gather" and show me the official usage |
| Re-authenticate | The docs session expired. Let me re-login |
