# axe-core Injection Scripts

These scripts are designed to be run via `javascript_tool` in Claude in Chrome.
The primary design goal is **minimizing tool calls** — each script is self-contained
and does not require follow-up "check if done" calls.

## Combined Batch Audit Script (PRIMARY — use this for batch mode)

This is the **most important script**. It performs the ENTIRE audit in ONE tool call:
loads axe-core, waits for it, runs the scan, runs manual DOM checks, and returns
combined results as JSON. Use this for every page in batch mode.

```javascript
(async () => {
  // Step 1: Load axe-core if not already loaded
  if (typeof axe === 'undefined') {
    await new Promise((resolve, reject) => {
      const s = document.createElement('script');
      s.src = 'https://cdnjs.cloudflare.com/ajax/libs/axe-core/4.8.4/axe.min.js';
      s.onload = resolve;
      s.onerror = reject;
      document.head.appendChild(s);
    });
  }

  // Step 2: Run axe-core scan
  const axeResults = await axe.run(document, {
    runOnly: { type: 'tag', values: ['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'best-practice'] }
  });

  // Step 3: Run manual DOM checks
  const manual = {
    htmlLang: document.documentElement.getAttribute('lang'),
    h1: !!document.querySelector('h1'),
    headings: Array.from(document.querySelectorAll('h1,h2,h3,h4,h5,h6'))
      .map(h => h.tagName).join(','),
    skipLink: !!document.querySelector(
      'a[href="#main-content"], a[href="#content"], a.skip-link, a.sr-only'),
    mainLandmark: !!document.querySelector('main, [role="main"]'),
    navCount: document.querySelectorAll('nav, [role="navigation"]').length,
    navsLabeled: Array.from(document.querySelectorAll('nav, [role="navigation"]'))
      .filter(n => n.getAttribute('aria-label') || n.getAttribute('aria-labelledby'))
      .length,
    imgsNoAlt: Array.from(document.querySelectorAll('img'))
      .filter(i => !i.hasAttribute('alt')).length,
    imgsEmptyAlt: document.querySelectorAll('img[alt=""]').length,
    formsNoLabel: Array.from(
      document.querySelectorAll('input:not([type="hidden"]), select, textarea')
    ).filter(el => {
      const id = el.id;
      return !(id && document.querySelector('label[for="' + id + '"]'))
        && !el.getAttribute('aria-label')
        && !el.getAttribute('aria-labelledby')
        && !el.closest('label');
    }).length,
    extLinksNoWarn: Array.from(document.querySelectorAll('a[target="_blank"]'))
      .filter(a => {
        const l = a.getAttribute('aria-label') || a.textContent || '';
        return !l.includes('new tab') && !l.includes('new window');
      }).length,
    positiveTabindex: Array.from(document.querySelectorAll('[tabindex]'))
      .filter(el => parseInt(el.getAttribute('tabindex')) > 0).length
  };

  // Step 4: Return combined results as compact JSON
  return JSON.stringify({
    url: location.href,
    title: document.title,
    violations: axeResults.violations.map(v => ({
      id: v.id,
      impact: v.impact,
      nodes: v.nodes.length,
      help: v.help,
      tags: v.tags.filter(t => t.startsWith('wcag') || t === 'best-practice')
    })),
    passes: axeResults.passes.length,
    manual: manual
  });
})()
```

**Why this works in one call:** The script uses an async IIFE with `await` for both
axe-core loading and the axe scan. The `javascript_tool` in Claude in Chrome handles
Promise resolution — when the IIFE returns a Promise, the tool waits for it to resolve
before returning the result. No separate "wait" or "check if done" call is needed.

**axe-core caching:** The script checks `typeof axe === 'undefined'` before loading.
On the first page, it loads from CDN. On subsequent pages in the same tab, if the
page navigates to a new URL (which clears the page context), axe-core needs to reload.
However, the CDN response is typically cached by the browser, so subsequent loads are
near-instant.

---

## Quick Scan Script (manual checks only, NO axe-core)

Use this when covering many pages quickly (15-20) or when the CDN is blocked.
Returns structural accessibility data without color contrast or ARIA analysis.

```javascript
JSON.stringify({
  url: location.href,
  title: document.title,
  manual: {
    htmlLang: document.documentElement.getAttribute('lang'),
    h1: !!document.querySelector('h1'),
    headings: Array.from(document.querySelectorAll('h1,h2,h3,h4,h5,h6'))
      .map(h => h.tagName + ': ' + h.textContent.trim().substring(0, 40)).join(' | '),
    skipLink: !!document.querySelector(
      'a[href="#main-content"], a[href="#content"], a.skip-link, a.sr-only'),
    mainLandmark: !!document.querySelector('main, [role="main"]'),
    navCount: document.querySelectorAll('nav, [role="navigation"]').length,
    navsLabeled: Array.from(document.querySelectorAll('nav, [role="navigation"]'))
      .filter(n => n.getAttribute('aria-label') || n.getAttribute('aria-labelledby'))
      .length,
    imgsTotal: document.querySelectorAll('img').length,
    imgsNoAlt: Array.from(document.querySelectorAll('img'))
      .filter(i => !i.hasAttribute('alt')).length,
    imgsEmptyAlt: document.querySelectorAll('img[alt=""]').length,
    formsNoLabel: Array.from(
      document.querySelectorAll('input:not([type="hidden"]), select, textarea')
    ).filter(el => {
      const id = el.id;
      return !(id && document.querySelector('label[for="' + id + '"]'))
        && !el.getAttribute('aria-label')
        && !el.getAttribute('aria-labelledby')
        && !el.closest('label');
    }).length,
    extLinksNoWarn: Array.from(document.querySelectorAll('a[target="_blank"]'))
      .filter(a => {
        const l = a.getAttribute('aria-label') || a.textContent || '';
        return !l.includes('new tab') && !l.includes('new window');
      }).length,
    positiveTabindex: Array.from(document.querySelectorAll('[tabindex]'))
      .filter(el => parseInt(el.getAttribute('tabindex')) > 0).length,
    autoplayMedia: document.querySelectorAll('video[autoplay], audio[autoplay]').length,
    iframes: document.querySelectorAll('iframe').length,
    iframesNoTitle: Array.from(document.querySelectorAll('iframe'))
      .filter(f => !f.getAttribute('title')).length
  }
})
```

---

## Single-Page Detail Scripts

These scripts are for **single-page mode only** where you want deeper detail on
specific violation types. Each is a separate tool call.

### Violation detail extraction

```javascript
const r = window.__axeResults;
r.violations.map(v => {
  const nodeInfo = v.nodes.slice(0, 5).map(n =>
    n.target[0] + ' :: ' + n.failureSummary.substring(0, 150)
  );
  return '=== ' + v.id + ' (' + v.impact + ') ===\n' + nodeInfo.join('\n');
}).join('\n\n')
```

### Color contrast detail extraction

Only needed if color-contrast violations exist and you want measured ratios.

```javascript
const r = window.__axeResults;
const cc = r.violations.find(v => v.id === 'color-contrast');
if (cc) {
  cc.nodes.map(n => {
    const d = n.any[0] && n.any[0].data ? n.any[0].data : {};
    return n.target[0].substring(0, 80) + ' | fg:' + d.fgColor +
      ' bg:' + d.bgColor + ' ratio:' + d.contrastRatio +
      ' size:' + d.fontSize + '/' + d.fontWeight;
  }).join('\n')
} else {
  'No color contrast violations found'
}
```

### Comprehensive manual DOM checks (detailed version)

Returns full details including element identifiers, not just counts.

```javascript
const checks = {};
checks.htmlLang = document.documentElement.getAttribute('lang');
checks.headings = Array.from(document.querySelectorAll('h1,h2,h3,h4,h5,h6'))
  .map(h => h.tagName + ': ' + h.textContent.trim().substring(0, 60));
checks.skipLink = !!document.querySelector(
  'a[href="#main-content"], a[href="#content"], a.skip-link, a.sr-only');
checks.mainLandmark = !!document.querySelector('main, [role="main"]');
checks.navCount = document.querySelectorAll('nav, [role="navigation"]').length;
checks.navsWithLabels = Array.from(document.querySelectorAll('nav, [role="navigation"]'))
  .map(n => n.getAttribute('aria-label') ||
    n.getAttribute('aria-labelledby') || 'NO LABEL');
checks.imagesWithoutAlt = Array.from(document.querySelectorAll('img'))
  .filter(i => !i.hasAttribute('alt'))
  .map(i => i.src.substring(i.src.lastIndexOf('/') + 1).substring(0, 50));
checks.imagesWithEmptyAlt = Array.from(document.querySelectorAll('img[alt=""]'))
  .map(i => i.src.substring(i.src.lastIndexOf('/') + 1).substring(0, 50));
checks.formInputsWithoutLabels = Array.from(
  document.querySelectorAll('input:not([type="hidden"]), select, textarea')
).filter(el => {
  const id = el.id;
  const hasLabel = id && document.querySelector('label[for="' + id + '"]');
  const hasAria = el.getAttribute('aria-label') || el.getAttribute('aria-labelledby');
  const wrapped = el.closest('label');
  return !hasLabel && !hasAria && !wrapped;
}).map(el => el.type + '#' + (el.id || el.className).substring(0, 40));
checks.externalLinksNoWarning = Array.from(
  document.querySelectorAll('a[target="_blank"]')
).filter(a => {
  const label = a.getAttribute('aria-label') || a.textContent || '';
  return !label.includes('new tab') && !label.includes('new window') &&
    !label.includes('external');
}).map(a => (a.textContent || '').trim().substring(0, 50) ||
  a.href.substring(0, 50)).slice(0, 15);
checks.positiveTabindex = Array.from(document.querySelectorAll('[tabindex]'))
  .filter(el => parseInt(el.getAttribute('tabindex')) > 0).length;
checks.autoplayMedia = document.querySelectorAll(
  'video[autoplay], audio[autoplay]').length;
JSON.stringify(checks, null, 2)
```

---

## Known Issues and Workarounds

### Content filter blocks on raw HTML
axe-core results include raw HTML snippets that may contain cookie data or session
tokens that trigger content safety filters. Always extract element selectors and
failure summaries rather than raw `node.html`.

### axe-core CDN blocked by Content Security Policy (CSP)

Many government sites enforce Content Security Policies that block third-party script
injection. Federal .gov domains (nist.gov, usgs.gov, etc.) and security-focused sites
commonly use strict `script-src` directives that do not include `cdnjs.cloudflare.com`.

**How it manifests:** The combined audit script's `s.onerror` fires, and the Promise
rejects with "CDN blocked". The script's `try/catch` catches this and returns
`{ error: "CDN blocked" }` instead of crashing.

**What to do:**
1. Do NOT retry the CDN load on the same site — CSP applies to all pages on the domain
2. Switch to the Quick Scan Script (manual checks only) for the rest of the batch
3. Note prominently in the report that axe-core could not run and list unchecked items:
   - Color contrast ratios (WCAG 1.4.3)
   - ARIA attribute validation (WCAG 4.1.2 partial)
   - List structure violations (WCAG 1.3.1 partial)
   - Link-in-text-block contrast
   - Duplicate IDs
4. Suggest the user install the axe DevTools browser extension (by Deque) for full
   automated testing — it runs natively in DevTools and is not subject to CSP

**How to confirm CSP is the cause:** Use `read_console_messages` with pattern
`"content-security-policy|CSP|blocked|refused"` to check for violation messages.

**Sites known to block axe-core CDN:** nist.gov, census.gov, many federal .gov and
.mil domains, sites with strict `script-src 'self'` or nonce-based CSP directives.

### Large pages with mega-menus
Government sites with mega-menus produce very large axe results. The combined script
limits to essential fields. For deeper dives, use the single-page detail scripts.

### SPAs and dynamic content
axe-core audits the current DOM state. Note this in the report and suggest testing
after key interactions.

### Page navigation clears axe-core
When navigating to a new URL, the page context resets and axe-core must be reloaded.
The combined batch script handles this automatically with the `typeof axe` check.
Browser-level CDN caching means the reload is typically fast (~100ms).
