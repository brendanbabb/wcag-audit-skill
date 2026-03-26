# WCAG Accessibility Audit Skill for Claude

A Claude skill that performs automated WCAG 2.1 AA accessibility audits of web pages using three complementary data sources: axe-core automated scanning, Chrome accessibility tree analysis, and manual DOM inspection.

A personal project by Brendan Babb.

## Features

- **4 audit modes**: Single-page deep dive, batch crawl (5-10 pages), before/after diff, and XLSX tracking export
- **3 data sources**: axe-core 4.8.4, Chrome accessibility tree, manual DOM checks
- **Tool call efficient**: 4 calls for a single page, 14 for a 5-page batch
- **SharePoint-aware**: Filters noise from empty web part zones, Google Translate widget clutter, and identifies template-level vs page-specific issues
- **Actionable output**: Every issue includes a specific fix suggestion, WCAG criterion mapping, and severity classification

## Quick start

### Claude.ai
1. Download the skill files from this repo
2. Go to Claude.ai → Settings → Features → Skills
3. Upload the `.skill` file
4. Install [Claude in Chrome](https://chromewebstore.google.com/) for full 3-source audits

### Claude Code
Copy the `wcag-audit/` directory into your skill folder.

## Example prompts

```
# Single-page deep dive
"Audit https://www.baltimorecity.gov/ for WCAG accessibility"

# Batch audit a section
"Run a batch WCAG audit on the top 5 pages of baltimorecity.gov"

# Before/after comparison
"Re-audit the Baltimore homepage and compare to the baseline"

# Export for leadership
"Export the audit results to a tracking spreadsheet"
```

## Skill files

| File | Description |
|------|-------------|
| `SKILL.md` | Core skill definition with all 4 modes, tool call budgets, scoring formula |
| `references/axe-core-scripts.md` | Combined audit script, quick scan script, detail extraction snippets |
| `references/wcag-quick-ref.md` | Most commonly violated WCAG criteria with fix patterns |

## Scoring

Each page starts at 100 and loses points per violation type:
- Critical: -15 (missing labels, no keyboard access)
- Serious: -8 (contrast failures, missing alt text)
- Moderate: -3 (missing landmarks, heading gaps)
- Minor: -1 (empty containers, inconsistent markup)

| Score | Status |
|-------|--------|
| 90-100 | Good |
| 70-89 | Needs work |
| 50-69 | Poor |
| < 50 | Failing |

## Limitations

- Automated tools catch ~30-40% of WCAG issues
- Color contrast measurements may not account for background images
- Keyboard navigation should be tested manually
- Screen reader testing with NVDA/JAWS/VoiceOver remains essential
- PDF and embedded content require separate audits

## Credits

- [axe-core](https://github.com/dequelabs/axe-core) by Deque Systems
- [WCAG 2.1](https://www.w3.org/WAI/WCAG21/quickref/) by W3C
- Inspired by [Santi Garces' Civic Analytics Skills](https://sgarcese.github.io/Civic-Analytics-Agent-Workflow-Claude-Skill/)
- Built with Claude by Anthropic

## License

MIT License
