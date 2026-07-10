---
name: wp-site-builder
description: >
  Use this skill whenever someone wants to build a website that will run on WordPress, whether they're starting from scratch, have a design brief, or want to convert an existing static site. Triggers on any request involving: building a WordPress theme, creating a personal site, building a portfolio or professional site, converting a design to WordPress, or building any content-managed website. This skill covers the full process from static HTML/CSS prototype through to a production-ready WordPress theme, with explicit standards for content management architecture, performance, and accessibility. Use it even if the user just says "build me a site" or "I need a WordPress theme" — do not skip phases or abbreviate the process.
---

# WordPress Site Builder

A complete process for building production-quality WordPress sites, from initial prototype through to a live, maintainable theme. Covers both the design/prototyping phase and the WordPress conversion phase, with detailed standards for each.

## Overview

The process has two phases:

1. **Static prototype** — Build the complete design in HTML/CSS/vanilla JS first. No WordPress, no frameworks.
2. **WordPress theme** — Convert the prototype to a fully functional WP theme using native WordPress patterns throughout.

Do not skip the prototype phase. It produces better themes because design decisions are resolved before the complexity of WordPress is introduced. It also gives the client something tangible to review before any CMS work begins.

Read the reference files when you reach each phase:
- `references/phase1-prototype.md` — Detailed standards for the static prototype
- `references/phase2-wordpress.md` — Complete WordPress theme architecture
- `references/content-management.md` — **Content management patterns — read this before writing any template**
- `references/performance-accessibility.md` — Performance, accessibility, and agent-readiness standards

---

## Phase 1: Static Prototype

**Before writing any code**, gather:
- Site structure (pages, sections, content types)
- Aesthetic direction (color palette, typography preferences, vibe/mood)
- A photo or headshot if it's a personal/portfolio site
- Any reference sites the person likes

Build the prototype as a set of HTML files with a shared CSS file and a single JS file. See `references/phase1-prototype.md` for full standards.

**Non-negotiables at prototype stage:**
- Vanilla JS only. No React, Vue, or JS frameworks of any kind.
- No CSS frameworks (no Tailwind, Bootstrap). Write the CSS from scratch.
- Semantic HTML throughout. Use `<nav>`, `<main>`, `<article>`, `<section>`, `<header>`, `<footer>` correctly.
- All pages must share the same nav, footer, fonts, and color palette.
- Dark mode toggle if the design calls for it — wire it to `localStorage` and `data-theme` attribute on `<html>`.
- CSS custom properties (`--variable-name`) for all colors, fonts, and spacing values. No magic numbers scattered through the stylesheet.
- Mobile-first or at minimum fully responsive. Check at 375px, 640px, 960px, and 1280px.

---

## Phase 2: WordPress Theme

After the prototype is approved, convert it to a WordPress theme. See `references/phase2-wordpress.md` for the full file structure and template patterns.

**Read `references/content-management.md` before writing a single template file.** Content management architecture is the decision that determines whether the theme is maintainable or not. Get it wrong and every piece of content requires a developer to change.

**Non-negotiables at WordPress stage:**
- Every piece of editable text must be editable from the WordPress admin without touching code.
- Use the correct WordPress mechanism for each type of content (see content management reference).
- Register ACF fields in code (`acf_add_local_field_group()`), not via the ACF UI. Fields registered in code are part of the theme and travel with it.
- Custom post types registered in `functions.php`, not via plugins.
- All assets enqueued via `wp_enqueue_style()` and `wp_enqueue_script()`, never hardcoded in templates.
- The theme must include `wp_head()` and `wp_body_open()` and `wp_footer()` in the correct locations.
- Featured images used as the native mechanism for page/post photography wherever possible.
- Add the agent-readiness fundamentals (feed discovery, theme-color/color-scheme meta, prefers-reduced-motion) — see "Agent Readiness" in `references/performance-accessibility.md`. Do this by default on every build, not just when a client asks.
- Add the full favicon/app-icon set and web app manifest (including the `favicon.ico`-at-docroot nginx gotcha) — see "Favicons, App Icons, and Web App Manifest" in `references/performance-accessibility.md`. Do this by default on every build.
- Self-host fonts by default rather than loading from Google Fonts' CDN; use `scrollbar-gutter: stable`, `dvh`/`svh` for full-viewport layouts, `text-wrap: balance`/`pretty`, and `font-size: 16px`+ on mobile form inputs — see "Performance" and "Accessibility" in `references/performance-accessibility.md`.

---

## Delivering the Theme

Deliver as a `.zip` file containing only the theme folder. The zip should install directly via Appearance → Themes → Add New → Upload.

Also deliver any WordPress XML import files (WXR format) as separate files for initial content population. These go to Tools → Import, not into the theme zip. Clearly document which file goes where.

---

## Reference Files

| File | When to read |
|------|-------------|
| `references/phase1-prototype.md` | Before writing prototype HTML/CSS |
| `references/phase2-wordpress.md` | Before converting to WordPress theme |
| `references/content-management.md` | Before writing any template — read this first |
| `references/performance-accessibility.md` | Before finalizing either phase — also covers favicons/app icons/web app manifest, agent-readiness (feed discovery, theme-color, reduced-motion, and when to reach for the standard plugins vs. hand-rolling), self-hosted fonts, and layout details (scrollbar-gutter, dvh/svh, text-wrap) |

---

## Updating This Skill

**At the end of every working session, always ask:** did we discover anything that should be added to this skill?

This skill is only valuable if it stays current. Information discovered in the course of a real build — bugs, workarounds, patterns that worked, patterns that didn't — is exactly what makes it useful for future projects. If that information isn't captured here, it's lost.

**What belongs in the skill (add it to the relevant reference file):**
- Hosting-specific constraints or gotchas (e.g. CDN caching behaviour on managed hosts)
- WordPress patterns that worked well or caused unexpected problems
- Performance or accessibility techniques discovered in practice
- CSS/JS patterns for specific UI challenges (mobile nav, modals, scroll animations)
- Security patterns and escaping rules
- Any "I wish I'd known this at the start" discoveries

**What does NOT belong in the skill:**
- Client-specific content (post copy, image URLs, specific page titles)
- One-off decisions that are unlikely to recur on other sites
- Version numbers of specific theme deliveries

**How to update:** Append new findings to the appropriate reference file under a clear `## Heading`. Don't edit existing content unless it's wrong — add new sections. Then repackage the skill zip and deliver it alongside the project docs.

The goal is that a future session building a different WordPress site should benefit from everything learned in every previous session.

---

