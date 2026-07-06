# Phase 1: Static Prototype Standards

## Purpose

The prototype resolves all design decisions before WordPress complexity is introduced. It must be complete enough that a client can review and approve the full design, and clean enough that converting it to a WordPress theme is straightforward.

## File Structure

```
prototype/
├── index.html          — Homepage
├── about.html          — About page
├── [page].html         — One file per page
├── assets/
│   ├── css/
│   │   └── main.css    — Single shared stylesheet
│   └── js/
│       └── main.js     — Single shared script
```

Keep it flat. One CSS file, one JS file, shared across all pages. This prevents the drift where nav looks slightly different on page 3 or the footer font changes on the contact page. That drift is a signal the prototype isn't ready for conversion.

## HTML Standards

- Use semantic elements: `<nav>`, `<main>`, `<article>`, `<section>`, `<header>`, `<footer>`, `<aside>`
- Every page must have a `<main id="main-content">` element
- Every page must have a skip link: `<a class="skip-link" href="#main-content">Skip to content</a>` as the first element in `<body>`
- Images must have descriptive `alt` attributes. Decorative images get `alt=""`
- `<nav>` elements must have `aria-label` attributes when there are multiple nav regions on the page (e.g. `aria-label="Primary"` for the main nav, `aria-label="Social links"` for footer links)
- Heading hierarchy must be logical. One `<h1>` per page. Don't skip levels.
- Interactive elements (buttons, links) must have accessible labels. Icon-only buttons need `aria-label`.
- SVG icons used decoratively must have `aria-hidden="true" focusable="false"`

## CSS Standards

**Custom properties for everything:**
```css
:root {
  --bg: #FAFAF8;
  --text: #0A0A0A;
  --accent: #FF3F00;
  --accent-text: #B52B00;  /* Darker version for text — must pass AA contrast */
  --border: rgba(0,0,0,0.09);
  --mono: 'IBM Plex Mono', monospace;
  --serif: 'Georgia', serif;
  --max: 1080px;
}
```

Always create a separate `--accent-text` variable for text uses of the accent color. The decorative accent and the accessible text color are almost never the same value. Check contrast ratios: 4.5:1 minimum for body text against its background (WCAG AA).

**Dark mode:**
```css
[data-theme="dark"] {
  --bg: #0D0D0D;
  --text: #F0EDE8;
  /* etc. */
}
```

Wire to `localStorage` and system preference in JS. Toggle via `data-theme` attribute on `<html>`, not by adding/removing a class on `<body>`.

**No magic numbers.** If a value appears more than once, it should be a variable. Spacing, border-radius, font sizes — all variables.

**Fonts:**

Load Google Fonts via `<link rel="preconnect">` hints plus a `<link rel="stylesheet">` tag, never via CSS `@import`. The `@import` approach blocks rendering because the browser must download the CSS before it can discover and request the fonts.

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...&display=swap">
```

Use `display=swap` (not `display=optional`). `optional` causes inconsistent rendering across pages when network conditions vary.

**Responsive:**
```css
.container {
  max-width: var(--max);
  margin: 0 auto;
  padding: 0 40px;
}

@media (max-width: 640px) {
  .container { padding: 0 20px; }
}
```

Check the nav at mobile widths. A fixed-height nav with all links in a single row will clip content on small screens. Plan for a stacked or scrollable nav at ≤640px. Do not use hamburger menus unless the client specifically requests one — they hide navigation and add interaction cost.

## JavaScript Standards

One file. Use IIFEs or function scope to avoid polluting the global namespace.

```js
/* main.js */

/* Dark mode — run immediately to avoid flash */
(function () {
  var saved = localStorage.getItem('theme');
  var prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
  if (saved === 'dark' || (!saved && prefersDark)) {
    document.documentElement.setAttribute('data-theme', 'dark');
  }
})();

document.addEventListener('DOMContentLoaded', function () {
  // everything else
});
```

Dark mode JS must run **before** `DOMContentLoaded` to prevent a flash of the wrong theme on page load.

Event listeners on `mousemove` or scroll should use `{ passive: true }` for performance.

Animation loops (`requestAnimationFrame`) should use `transform` and `opacity` for GPU-composited changes. Do not animate `top`, `left`, `width`, or `height` — these trigger layout recalculation.

## Images

All images used in the prototype should be at the correct display size, not larger. A headshot displayed at 400px wide does not need to be a 1200px file. Over-sized images are the single most common cause of poor PageSpeed scores.

Rule of thumb: serve at 2× the maximum display width for retina screens. A 400px display width → 800px image file.

Add `width` and `height` attributes to all `<img>` tags so the browser can reserve layout space before the image loads:
```html
<img src="headshot.jpg" alt="..." width="800" height="640">
```

For the LCP image (the largest image visible on initial load — usually a hero photo), add `fetchpriority="high"`:
```html
<img src="hero.jpg" alt="..." width="800" height="640" fetchpriority="high">
```

## Completeness Checklist Before Conversion

Before converting to WordPress, confirm:
- [ ] All pages render correctly and consistently
- [ ] Dark mode works on all pages
- [ ] Nav is correct on all pages — same structure, same styling
- [ ] Footer is correct on all pages
- [ ] Fully responsive — tested at 375px, 640px, 960px
- [ ] No page-specific CSS or JS (everything is in shared files)
- [ ] All images sized appropriately
- [ ] No console errors
- [ ] Semantic HTML with correct heading hierarchy
- [ ] Skip link present and functional
- [ ] Color contrast passes AA for all text

---

