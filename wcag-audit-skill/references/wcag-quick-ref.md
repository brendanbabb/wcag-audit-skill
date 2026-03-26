# WCAG 2.1 AA Quick Reference — Common Violations and Fix Patterns

This reference covers the WCAG 2.1 Level A and AA success criteria most frequently
violated on municipal and government websites. Consult when writing fix suggestions.

## Perceivable (Principle 1)

### 1.1.1 Non-text Content (A)
**Requirement**: All non-text content has a text alternative.
**Common violations**:
- Images missing `alt` attribute entirely
- Decorative images that should have `alt=""` but don't
- Icon links (social media, nav) with no accessible name
- CAPTCHAs without audio alternative
**Fix patterns**:
- Informative image: `<img alt="Mayor Suzanne LaFrance official portrait">`
- Decorative image: `<img alt="" aria-hidden="true">`
- Icon link: `<a href="..." aria-label="Follow us on Facebook"><img alt="" ...></a>`
- Linked image: Alt text describes the link destination, not the image appearance

### 1.3.1 Info and Relationships (A)
**Requirement**: Information, structure, and relationships conveyed visually are
available programmatically.
**Common violations**:
- Heading hierarchy skips levels (H1 → H3, or no H1)
- Lists using divs/spans instead of ul/ol/li
- Data tables without proper th/scope/caption
- Form labels not associated with inputs
- ARIA attributes on invalid elements
- No landmark structure (main, nav, complementary)
**Fix patterns**:
- Headings: Ensure H1 exists, levels don't skip (H1 → H2 → H3)
- Landmarks: Wrap content in `<main>`, nav in `<nav aria-label="...">`, sidebar in `<aside>`
- Lists: Replace div-based lists with semantic ul/li
- Tables: Add `<th scope="col">` for headers, `<caption>` for table purpose

### 1.4.3 Contrast Minimum (AA)
**Requirement**: Text has contrast ratio of at least 4.5:1 (normal text) or 3:1
(large text: 18pt or 14pt bold).
**Common violations**:
- Light gray text on white backgrounds
- White text on medium-tone colored backgrounds
- Placeholder text with insufficient contrast
- Link text that relies on color alone to distinguish from body text
**Fix patterns**:
- Identify the exact ratio using axe-core's contrast data
- Suggest specific hex values that meet the threshold
- For nav bars: darkening the background is usually better than darkening text
- Calculator: target ratio = 4.5 for normal text, 3.0 for ≥18pt or ≥14pt bold

### 1.4.11 Non-text Contrast (AA) — WCAG 2.1
**Requirement**: UI components and graphical objects have 3:1 contrast ratio.
**Common violations**:
- Form field borders too light to see
- Icons without sufficient contrast
- Chart elements indistinguishable
**Fix patterns**:
- Ensure form borders are at least 3:1 against the background
- Add visible focus indicators with sufficient contrast

## Operable (Principle 2)

### 2.1.1 Keyboard (A)
**Requirement**: All functionality is operable via keyboard.
**Common violations**:
- Click handlers on non-focusable elements (div, span)
- Custom widgets without keyboard support
- Dropdown menus only work on hover
- Focus traps in modals or iframes
**Fix patterns**:
- Use native interactive elements (button, a) instead of div with onclick
- Add `tabindex="0"` and keyboard event handlers to custom widgets
- Ensure modals trap focus correctly (cycle within modal, escape to close)

### 2.4.1 Bypass Blocks (A)
**Requirement**: Mechanism to bypass repeated content blocks.
**Common violations**:
- No skip navigation link
- Skip link present but hidden and not focusable
- Skip link target doesn't exist
**Fix patterns**:
- Add as first focusable element:
  `<a href="#main-content" class="sr-only sr-only-focusable">Skip to main content</a>`
- Add `id="main-content"` to the main content container
- CSS: `.sr-only` visually hidden, `.sr-only-focusable:focus` becomes visible

### 2.4.3 Focus Order (A)
**Requirement**: Focus order preserves meaning and operability.
**Common violations**:
- CSS visual reordering without DOM reordering
- Positive tabindex values overriding natural order
- Modal content behind the modal in tab order
**Fix patterns**:
- Match DOM order to visual order
- Use `tabindex="0"` (not positive values) for custom focusable elements
- Remove all `tabindex` values greater than 0

### 2.4.4 Link Purpose in Context (A)
**Requirement**: Link purpose can be determined from link text or context.
**Common violations**:
- "Click here", "Read more", "Learn more" with no context
- Image links with no alt text
- Links that open PDFs or external sites without indication
**Fix patterns**:
- Replace "Read more" with "Read more about the housing strategy"
- Image links: `alt` describes the destination
- External links: append "(opens in new tab)" to aria-label
- PDF links: append "(PDF)" to link text

### 2.4.6 Headings and Labels (AA)
**Requirement**: Headings and labels describe topic or purpose.
**Common violations**:
- Generic headings ("More information")
- Form labels that don't describe the expected input

## Understandable (Principle 3)

### 3.1.1 Language of Page (A)
**Requirement**: Default human language can be programmatically determined.
**Common violations**:
- Missing `lang` attribute on `<html>` element
- Wrong language code
**Fix patterns**:
- Add `<html lang="en">` (or appropriate language code)
- For multilingual content, use `lang` attribute on specific elements

### 3.2.5 Change on Request (AAA — but common best practice)
**Requirement**: Changes of context are initiated only by user request.
**Common violations**:
- Links opening new windows/tabs without warning
- Auto-submitting forms
**Fix patterns**:
- Add `aria-label` including "(opens in new tab)" for `target="_blank"` links
- Or add a visually hidden `<span class="sr-only">(opens in new tab)</span>`

## Robust (Principle 4)

### 4.1.2 Name, Role, Value (A)
**Requirement**: All UI components have accessible name, role, and state.
**Common violations**:
- Buttons with no text or aria-label
- Custom widgets without ARIA roles
- Form inputs without labels
- ARIA attributes used on elements with invalid roles
**Fix patterns**:
- Buttons: Add `aria-label` or visible text content
- Form inputs: Associate with `<label for="...">` or add `aria-label`
- Custom widgets: Add appropriate `role`, `aria-expanded`, `aria-selected`, etc.
- Invalid ARIA: Add `role="region"` to elements using `aria-labelledby`, or remove the ARIA attribute

## Template-Level vs Page-Level Fixes

When reporting fixes, always classify:

**Template-level** (fix once, affects all pages):
- Skip navigation links
- Main landmark wrapper
- Navigation aria-labels
- Search input/button labels
- Color contrast on global nav
- Footer landmark
- Empty web part zone handling
- lang attribute on html element

**Page-level** (fix per page):
- Image alt text
- Heading hierarchy (H1 specifically)
- Link text clarity
- Form-specific labels
- Content-specific ARIA

Prioritize template-level fixes because they have the highest ROI — one change
can remediate hundreds of pages simultaneously.
