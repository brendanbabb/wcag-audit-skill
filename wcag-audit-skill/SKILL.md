---
name: wcag-audit
description: >
  Audit web pages for WCAG 2.1 AA accessibility compliance using axe-core, Chrome
  accessibility tree analysis, and manual DOM inspection. Supports single-page audit,
  batch crawl of multiple URLs with a rolled-up scorecard, before/after diff to measure
  remediation progress, and tracking spreadsheet export for leadership reporting.
  Use whenever someone asks about accessibility, ADA compliance, WCAG compliance,
  Section 508, screen reader issues, or says things like "audit this page", "check
  these URLs for accessibility", "compare before and after", "track our accessibility
  scores", or references WCAG criteria by number. Also triggers for building accessibility
  QA tools or checking government websites. Requires Claude in Chrome for full audit;
  partial audit available via web_fetch if browser tools are unavailable.
---

# WCAG 2.1 AA Accessibility Audit Skill

## Overview

This skill performs comprehensive accessibility audits of web pages using up to three
complementary data sources, then synthesizes findings into prioritized, actionable
reports with specific fix suggestions.

**Four operating modes:**

1. **Single-page audit** (default) — Deep audit of one URL with full issue report
2. **Batch crawl** — Audit multiple pages on a domain with a rolled-up scorecard,
   identifying template-level vs page-specific issues
3. **Before/after diff** — Compare two audit runs to measure remediation progress,
   with narrative summaries suitable for leadership reporting
4. **Tracking spreadsheet** — Export results to a structured XLSX file for
   longitudinal tracking, team handoff, and compliance reporting

**The three audit data sources are:**

1. **axe-core automated scan** — Inject the axe-core library into the live page and
   run it against WCAG 2.0 A/AA, WCAG 2.1 A/AA, and best-practice rules.

2. **Accessibility tree analysis** — Use `read_page` with `filter: "all"` to get the
   Chrome accessibility tree. Reveals missing landmarks, heading hierarchy gaps,
   unlabeled navigations, and structural issues.

3. **Manual DOM inspection** — Use `javascript_tool` to check for skip links, `lang`
   attribute, images without `alt`, `target="_blank"` links without warnings, form
   inputs without labels, and positive `tabindex` values.

## CRITICAL: Tool Call Budget Awareness

**Every tool call costs against a per-conversation limit.** The skill MUST minimize
tool calls, especially in batch mode. Failure to do so causes conversations to hit
limits mid-audit, requiring the user to say "continue" repeatedly.

**Tool call budget per mode:**
- Single-page audit: ~6-8 tool calls (acceptable)
- Batch audit: ~2 tool calls per page + ~3 for setup/report (target)
- Before/after diff: ~12-16 tool calls total (two single-page audits + comparison)

**Key efficiency rules:**
1. ALWAYS combine axe-core load + scan + manual checks into ONE javascript_tool call
2. In batch mode, SKIP the read_page accessibility tree (it's huge and redundant with
   axe-core for violation counting)
3. NEVER use separate tool calls for "wait then check if done" — use the combined
   script with built-in waiting
4. Check if axe-core is already loaded before re-injecting it

## Prerequisites

**Required:**
- Claude in Chrome browser tools (read_page, javascript_tool, navigate, tabs_context_mcp)
- The target page must be open in a Chrome tab in the MCP tab group

**Fallback mode (partial audit):**
- If browser tools are unavailable, use `web_fetch` to get page content
- Note in the report that the audit is partial and which checks could not be performed

## Handling Content Security Policy (CSP) Blocking

Some government and security-conscious sites (e.g. nist.gov, many federal .gov domains)
enforce Content Security Policies that block injection of third-party scripts like
axe-core from cdnjs.cloudflare.com. When this happens:

1. **The combined audit script will catch the error** — the `try/catch` wrapper in
   the combined batch script returns `{ error: "CDN blocked" }` instead of crashing.

2. **Automatically fall back to quick scan mode.** Run the manual DOM checks script
   (no external CDN dependency) instead. This still catches: missing skip links,
   missing main landmark, unlabeled navs, images without alt, form inputs without
   labels, heading hierarchy, external links without new-tab warnings, lang attribute,
   positive tabindex, iframes without titles.

3. **Note in the report** that axe-core could not load due to the site's CSP, and
   list what could NOT be checked: color contrast ratios, ARIA attribute validation,
   list structure, link-in-text-block contrast, and other axe-core-only rules.

4. **Do NOT retry the CDN load** on subsequent pages of the same site — if the CSP
   blocks it on the first page, it will block it on all pages. Switch to quick scan
   for the entire batch.

**How to detect CSP blocking:** The combined audit script uses a Promise-based loader.
If `s.onerror` fires (or the Promise rejects with "CDN blocked"), the CSP is active.
You can also check the browser console for CSP violation messages using
`read_console_messages` with pattern `"content-security-policy|CSP|blocked"`.

**Sites known to block axe-core CDN injection:**
- nist.gov (federal CSP)
- census.gov (federal CSP)
- Many other .gov and .mil domains
- Sites using strict `script-src` directives without `cdnjs.cloudflare.com`

**Alternative for full axe-core audits on CSP-restricted sites:** Suggest the user
install the axe DevTools browser extension (by Deque), which runs axe-core natively
within DevTools and is not subject to CSP restrictions. The skill's manual checks
still complement the extension's findings.

## Mode 1: Single-Page Audit Workflow

### Step 1: Establish browser context

Get the tab context and verify the target page is loaded.

```
Use tabs_context_mcp to find available tabs.
If the target URL is not already open, navigate to it.
Record the tabId for all subsequent tool calls.
```

### Step 2: Pull the accessibility tree

**Only in single-page mode.** Skip this step entirely in batch mode.

```
Use read_page with:
  - depth: 8 (deeper if the page is complex)
  - filter: "all"
  - tabId: <target tab>
```

From the tree, extract and note:
- **Heading hierarchy**: List all headings (H1-H6) and their text. Flag if H1 is
  missing or levels are skipped.
- **Landmarks**: Count `navigation`, `main`, `complementary`, `banner`, `contentinfo`
  landmarks. Flag if main is missing. Check if nav elements have aria-labels.
- **Unlabeled interactive elements**: Links, buttons, inputs that show as `link [ref_N]`
  with no accessible name.
- **Empty containers**: Named containers that are empty but visible in the tree.

### Step 3: Run combined axe-core + manual checks (ONE tool call)

**This is the most important efficiency optimization.** Use the combined audit script
from `references/axe-core-scripts.md` (Script: Combined Audit) that performs ALL of
the following in a single javascript_tool call:

1. Loads axe-core from CDN (with built-in wait)
2. Runs the axe-core scan
3. Runs all manual DOM checks
4. Returns everything as one JSON blob

See `references/axe-core-scripts.md` for the exact script. The key is that it uses
a Promise-based approach to load axe-core, wait for it, run the scan, run manual
checks, and return combined results — all in one execution.

### Step 4: Synthesize and present findings

Merge findings from all sources, deduplicating where they overlap. For each finding:

1. **Classify severity** using axe-core's impact levels:
   - **Critical**: Blocks access entirely (missing form labels, no keyboard access)
   - **Serious**: Major barrier (contrast failures, missing alt text, broken ARIA)
   - **Moderate**: Significant but not blocking (missing landmarks, heading gaps)
   - **Minor**: Polish issues (inconsistent markup, empty containers)

2. **Tag the source**: axe-core, a11y tree, manual, or multiple sources.

3. **Map to WCAG criterion**: Include the specific WCAG 2.1 success criterion number
   and level (A, AA, AAA). Common mappings:
   - 1.1.1 Non-text Content (A) — images, icons, media
   - 1.3.1 Info and Relationships (A) — headings, lists, landmarks, ARIA
   - 1.4.3 Contrast Minimum (AA) — text color contrast
   - 2.1.1 Keyboard (A) — keyboard accessibility
   - 2.4.1 Bypass Blocks (A) — skip links
   - 2.4.4 Link Purpose (A) — descriptive link text
   - 3.1.1 Language of Page (A) — lang attribute
   - 4.1.2 Name, Role, Value (A) — form labels, button names, ARIA

4. **Write a specific fix suggestion** for each issue. Fixes should be concrete,
   scoped, and prioritized.

### Step 5: Present the report

Present findings using the Visualizer tool as an interactive dashboard with:

- **Summary metrics**: Count of critical, serious, moderate, minor issues + passing
- **Category filters**: Buttons to filter by category
- **Issue cards**: Title, severity badge, source badge(s), WCAG criterion, description,
  affected elements, and fix suggestion
- **Methodology footer**: Note the audit sources and their scope

Use severity colors:
- Critical: `var(--color-background-danger)` / `var(--color-text-danger)`
- Serious: `#FAEEDA` / `#854F0B` (amber)
- Moderate: `var(--color-background-info)` / `var(--color-text-info)`
- Minor: `var(--color-background-secondary)` / `var(--color-text-secondary)`

Use source badges:
- axe-core: green badge (`#E1F5EE` / `#0F6E56`)
- a11y tree: blue badge (`#E6F1FB` / `#185FA5`)
- manual: purple badge (`#EEEDFE` / `#534AB7`)

## Important Caveats to Include in Every Report

Always note these limitations:

1. **Automated tools catch ~30-40% of WCAG issues**. The remaining issues require
   human judgment.
2. **Color contrast measurements** depend on computed styles and may not account for
   background images, gradients, or dynamic state changes.
3. **Keyboard navigation** should be manually tested by tabbing through the page.
4. **Screen reader testing** with actual assistive technology is essential.
5. **PDF and embedded content** accessibility requires separate audits.

## Handling Common Platforms

### SharePoint (Classic/2013/2016/2019)

- Empty web part zones clutter the a11y tree
- `aria-labelledby` on divs with no ARIA role
- Master page controls global nav, search, breadcrumbs, footer — flag template-level
  fixes separately
- Google Translate widget injects hundreds of generic focusable elements into the
  accessibility tree — note this as noise, not a real issue

### WordPress

- Check theme-generated markup for heading hierarchy
- Plugin-generated forms often lack proper labels

### React/SPA

- Dynamic content may not be present when axe-core runs
- Route changes may not announce to screen readers

---

## Mode 2: Batch Crawl Audit

### CRITICAL: Efficiency-First Design

Batch mode is designed to audit multiple pages within a tight tool call budget.
It deliberately trades depth for breadth compared to single-page mode:

- **NO accessibility tree analysis** (read_page) — too large per page, redundant
  with axe-core for violation counting
- **ONE combined audit tool call per page** after navigation
- **NO separate `wait` calls** — the combined script handles timing internally
- **Total target: 2 tool calls per page** (navigate + combined audit script)

### Page limits

- **Default: 5 pages**
- User can override: "audit up to 10 pages"
- **Hard cap: 10 pages** regardless of user request
- If user asks for more than 10, explain: "I'll audit 10 pages to stay within tool
  call limits. For broader coverage, we can run additional batches in follow-up
  conversations, or I can do a quick structural scan (no axe-core) of more pages."

If the discovered URL list exceeds the limit, present the full list and ask the user
to confirm or select pages. Always show the total discovered count.

### Quick scan mode (optional, for 10+ pages)

If the user asks for a quick scan, structural scan, or wants to cover many pages fast,
skip axe-core entirely and only run the manual DOM checks script. This takes ONE tool
call per page (no CDN load needed) and can cover 15-20 pages within budget.

Quick scan catches: missing skip links, missing main landmark, unlabeled navs, images
without alt, form inputs without labels, heading hierarchy issues, external links
without warnings, positive tabindex, lang attribute.

Quick scan does NOT catch: color contrast ratios, ARIA attribute misuse, list structure
violations, or other axe-core-only rules. Note these gaps in the report.

### Batch workflow

1. **Collect the URL list** (1 tool call). Use `javascript_tool` to extract internal
   links from the starting page:

   ```javascript
   Array.from(document.querySelectorAll('a[href]'))
     .map(a => a.href)
     .filter(h => h.startsWith('https://TARGET_DOMAIN/TARGET_PATH'))
     .filter(h => !h.includes('#') && !h.endsWith('.pdf') && !h.endsWith('.doc'))
     .filter((v, i, a) => a.indexOf(v) === i)
     .slice(0, 20)
   ```

   Apply the page limit and present to the user for confirmation.

2. **Audit each page** (2 tool calls per page). For each URL:
   - `navigate` to the page (1 tool call)
   - Run the **Combined Batch Audit** script from `references/axe-core-scripts.md`
     (1 tool call) — this loads axe, runs scan, runs manual checks, returns JSON
   - Store results in conversation context (no tool call needed)
   - Report brief progress inline: "Audited 3/5 pages..."

   **DO NOT** use `computer:wait` between navigate and the audit script.
   **DO NOT** use `read_page` in batch mode.
   **DO NOT** use separate tool calls for "load axe", "run axe", "extract results",
   "run manual checks" — the combined script does ALL of this in ONE call.

3. **Identify site-wide patterns.** After all pages, analyze which violations appear
   on 80%+ of pages — these are template-level issues.

4. **Present a rolled-up scorecard** using the Visualizer (1 tool call).

5. **Offer to export** to a tracking spreadsheet (Mode 4).

**Example tool call budget for a 5-page batch:**
- tabs_context_mcp: 1
- javascript_tool (discover URLs): 1
- navigate + combined audit × 5 pages: 10
- Visualizer report: 1
- **Total: 13 tool calls** ✓

**Example tool call budget for a 10-page batch:**
- tabs_context_mcp: 1
- javascript_tool (discover URLs): 1
- navigate + combined audit × 10 pages: 20
- Visualizer report: 1
- **Total: 23 tool calls** ✓

### Scoring formula

- Start at 100
- Subtract 15 per critical violation type (per unique violation ID, not per element)
- Subtract 8 per serious violation type
- Subtract 3 per moderate violation type
- Subtract 1 per minor violation type
- Floor at 0

Score interpretation:
- 90-100: Good (minor issues only)
- 70-89: Needs work (some serious issues)
- 50-69: Poor (multiple serious/critical issues)
- Below 50: Failing (significant barriers to access)

### Batch report Visualizer template

Present as an interactive HTML with chart and data table:

```
[Site scorecard header]
[Summary metrics: pages audited, average score, template issues, worst page]
[Horizontal bar chart: scores by page, color-coded by status]
[Data table: page, section, score, status, critical/serious/moderate/minor counts]
[Template-level issues: cards with severity, WCAG criterion, fix suggestions]
[Page-specific issues: worst offenders highlighted]
[Methodology footer with export button via sendPrompt]
```

---

## Mode 3: Before/After Diff

Compare two audit runs of the same page to measure remediation progress.

### Workflow

1. **Establish baseline.** Either use results from earlier in the conversation,
   or run a fresh single-page audit as baseline.
2. **Run the current audit.** Full single-page audit on the current page state.
3. **Compute the diff.** Compare violation IDs between baseline and current:
   - **Resolved**: in baseline, NOT in current → green
   - **New**: in current, NOT in baseline → red
   - **Persisting**: in both → amber
   - **Improved**: same ID but fewer affected nodes → blue
4. **Present the diff report** using Visualizer with score change, progress metrics.
5. **Generate a narrative summary** suitable for pasting into email or report.

---

## Mode 4: Tracking Spreadsheet

Generate an XLSX file that logs accessibility scores per page over time.

### Workflow

1. Collect audit data from the current conversation.
2. Read `/mnt/skills/public/xlsx/SKILL.md` for xlsx creation patterns.
3. Generate the file with these sheets:

   **Sheet 1: Score Summary** — URL, title, date, score, severity counts, status
   **Sheet 2: All Issues** — URL, issue ID, severity, WCAG criterion, description,
   fix suggestion, source, scope (template/page), status (open)
   **Sheet 3: Template-Level Issues** (if batch) — issues on 80%+ of pages
   **Sheet 4: Audit Metadata** — date, tool version, standard, methodology

4. Present the file using `present_files`.

---

## Fallback Mode (No Browser Tools)

If Claude in Chrome is not available, perform a partial audit:

1. Use `web_fetch` to get page content
2. Analyze the markdown/text for: heading structure, link text quality, image
   references without alt text indicators, form structure
3. Note prominently that this is a **partial audit** and which checks could not
   be performed (contrast measurement, a11y tree, axe-core scan, keyboard checks)
4. Recommend the user connect Claude in Chrome for a full audit

---

## Sharing the Skill

### For Claude.ai users (via .skill file)
Upload the `.skill` file in Claude.ai settings → Features → Skills.

### For Claude Code users
Copy the `wcag-audit/` directory into the user's skill directory.

### For teams
Share via Google Drive or SharePoint. Note that Claude in Chrome is needed for full audits.

## Quick Reference: WCAG 2.1 AA

Read `references/wcag-quick-ref.md` for a concise mapping of the most commonly
violated criteria, their requirements, and typical fix patterns.
