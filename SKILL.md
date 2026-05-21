---
name: wp-site-builder
description: >
  Use this skill whenever someone wants to build a website that will run on WordPress, whether they're starting from scratch, have a design brief, or want to convert an existing static site. Triggers on any request involving: building a WordPress theme, creating a personal site, building a portfolio or professional site, converting a design to WordPress, or building any content-managed website. This skill covers the full process from static HTML/CSS prototype through to a production-ready WordPress theme, with explicit standards for content management architecture, performance, and accessibility.
---

# WordPress Site Builder

A complete process for building production-quality WordPress sites, from initial prototype through to a live, maintainable theme. Covers both the design/prototyping phase and the WordPress conversion phase, with detailed standards for each.

## Overview

The process has two phases:

1. **Static prototype** — Build the complete design in HTML/CSS/vanilla JS first. No WordPress, no frameworks.
2. **WordPress theme** — Convert the prototype to a fully functional WP theme using native WordPress patterns throughout.

Do not skip the prototype phase. It produces better themes because design decisions are resolved before the complexity of WordPress is introduced.

---

# Phase 1: Static Prototype

## Purpose

The prototype resolves all design decisions before WordPress complexity is introduced. It must be complete enough that a client can review and approve the full design, and clean enough that converting it to a WordPress theme is straightforward.

**Before writing any code**, gather:
- Site structure (pages, sections, content types)
- Aesthetic direction (color palette, typography preferences, vibe/mood)
- A photo or headshot if it's a personal/portfolio site
- Any reference sites the person likes

## File structure

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

One CSS file, one JS file, shared across all pages.

## Non-negotiables at prototype stage

- Vanilla JS only. No React, Vue, or JS frameworks.
- No CSS frameworks (no Tailwind, Bootstrap). Write the CSS from scratch.
- Semantic HTML throughout. Use `<nav>`, `<main>`, `<article>`, `<section>`, `<header>`, `<footer>` correctly.
- All pages must share the same nav, footer, fonts, and color palette.
- CSS custom properties (`--variable-name`) for all colors, fonts, and spacing values.
- Mobile-first or at minimum fully responsive. Check at 375px, 640px, 960px, and 1280px.

## CSS standards

**Custom properties for everything:**
```css
:root {
  --bg: #FAFAF8;
  --text: #0A0A0A;
  --accent: #3366FF;
  --accent-text: #2244AA;  /* Darker version for text — must pass AA contrast */
  --border: rgba(0,0,0,0.09);
  --font-sans: 'Inter', sans-serif;
  --font-mono: 'IBM Plex Mono', monospace;
  --max: 1080px;
}
```

Always create a separate `--accent-text` variable for text uses of the accent color. The decorative accent and the accessible text color are almost never the same value. Check contrast ratios: 4.5:1 minimum for body text (WCAG AA).

**Dark mode:**
```css
[data-theme="dark"] {
  --bg: #0D0D0D;
  --text: #F0EDE8;
}
```

Wire to `localStorage` and system preference in JS. Toggle via `data-theme` attribute on `<html>`, not a class on `<body>`.

**Fonts:** Load Google Fonts via `<link rel="preconnect">` hints plus a `<link rel="stylesheet">` tag, never via CSS `@import`. Use `display=swap`.

## JavaScript standards

One file. Dark mode JS must run **before** `DOMContentLoaded` to prevent a flash of the wrong theme:

```js
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

Event listeners on `mousemove` or scroll should use `{ passive: true }`. Animate only `transform` and `opacity` for GPU-composited changes.

## Images

Serve at 2x the maximum display width for retina screens. Add `width` and `height` attributes to all `<img>` tags. For the LCP image, add `fetchpriority="high"`.

## Completeness checklist before conversion

- [ ] All pages render correctly and consistently
- [ ] Dark mode works on all pages
- [ ] Nav and footer identical on all pages
- [ ] Fully responsive at 375px, 640px, 960px
- [ ] No page-specific CSS or JS
- [ ] All images sized appropriately
- [ ] Semantic HTML with correct heading hierarchy
- [ ] Skip link present and functional
- [ ] Color contrast passes AA for all text

---

# Phase 2: WordPress Theme

## Non-negotiables at WordPress stage

- Every piece of editable text must be editable from the WordPress admin without touching code.
- Use the correct WordPress mechanism for each type of content (see Content Management below).
- Register ACF fields in code (`acf_add_local_field_group()`), not via the ACF UI.
- Custom post types registered in `functions.php`, not via plugins.
- All assets enqueued via `wp_enqueue_style()` and `wp_enqueue_script()`, never hardcoded in templates.
- The theme must include `wp_head()`, `wp_body_open()`, and `wp_footer()` in the correct locations.

## Theme file structure

```
theme-name/
├── style.css               — Theme declaration (required)
├── functions.php           — All PHP: setup, CPTs, ACF, enqueue, helpers
├── header.php              — <html>, <head>, wp_head(), nav
├── footer.php              — footer markup, wp_footer()
├── index.php               — Fallback template (required)
├── front-page.php          — Homepage (when static front page is set)
├── page.php                — Generic page fallback
├── page-about.php          — About page (auto-applied by filename)
├── single.php              — Single blog post
├── archive.php             — Post archive / blog listing
├── 404.php                 — Not found page
├── template-parts/         — Reusable partial templates
└── assets/
    ├── css/main.css
    ├── js/main.js
    └── images/
```

## functions.php structure

Organize in clearly labelled sections:

```php
<?php
/* =============================================
   THEME SETUP
   ============================================= */
function theme_setup() {
    add_theme_support( 'title-tag' );
    add_theme_support( 'post-thumbnails' );
    add_theme_support( 'html5', [ 'search-form', 'comment-form', 'comment-list', 'gallery', 'caption', 'script', 'style' ] );
    register_nav_menus( [ 'primary' => 'Primary Navigation' ] );
}
add_action( 'after_setup_theme', 'theme_setup' );

function theme_content_width() {
    $GLOBALS['content_width'] = 700;
}
add_action( 'after_setup_theme', 'theme_content_width', 0 );

/* =============================================
   ENQUEUE ASSETS
   ============================================= */
/* =============================================
   CUSTOM POST TYPES
   ============================================= */
/* =============================================
   ACF FIELDS
   ============================================= */
/* =============================================
   HELPER FUNCTIONS
   ============================================= */
```

## Enqueuing assets

```php
function theme_enqueue_assets() {
    $ver = wp_get_theme()->get( 'Version' );

    wp_enqueue_style( 'theme-main',
        get_template_directory_uri() . '/assets/css/main.css',
        [], $ver );

    wp_enqueue_script( 'theme-main',
        get_template_directory_uri() . '/assets/js/main.js',
        [], $ver,
        [ 'in_footer' => true, 'strategy' => 'defer' ]
    );
}
add_action( 'wp_enqueue_scripts', 'theme_enqueue_assets' );

// Dequeue unused block library CSS
add_action( 'wp_enqueue_scripts', function() {
    wp_dequeue_style( 'wp-block-library' );
    wp_dequeue_style( 'wp-block-library-theme' );
    wp_dequeue_style( 'global-styles' );
}, 100 );
```

Always read the version dynamically — never hardcode it. The version string is appended as a `?ver=` query parameter to CSS and JS URLs for cache busting.

## header.php

```php
<!DOCTYPE html>
<html <?php language_attributes(); ?>>
<head>
<meta charset="<?php bloginfo( 'charset' ); ?>">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<?php wp_head(); ?>
</head>
<body <?php body_class(); ?>>
<?php wp_body_open(); ?>

<a class="skip-link" href="#main-content">Skip to content</a>

<nav class="site-nav" aria-label="Primary">
  <!-- navigation -->
</nav>
```

`wp_head()` and `wp_body_open()` are both required. The skip link must be the first focusable element.

## Template pages

Always call `the_post()` at the top so native WP functions work:

```php
<?php get_header(); the_post(); ?>
<main id="main-content">
  <h1><?php the_title(); ?></h1>
  <div><?php the_content(); ?></div>
</main>
<?php get_footer(); ?>
```

`<main id="main-content">` must be present on every template — it's where the skip link points.

---

# Content Management Architecture

**Read this before writing any WordPress template.**

The single most important architectural decision in a WordPress theme is: where does the content live? Get this wrong and the site looks right on launch but becomes unmaintainable immediately after — every content update requires a developer.

## The golden rule

**Never hardcode content in template files.**

This includes: page titles, introductory paragraphs, descriptions, headings, taglines, bio text, photo URLs, social media links, contact information — anything a site owner might want to update.

## The content management matrix

| Content type | Where it lives | How to edit |
|---|---|---|
| Page title | WordPress page title field | Page editor |
| Page body text | WordPress content editor | Page editor |
| Page featured photo | WordPress featured image | Page editor sidebar |
| Structured repeating data | ACF fields on the page | Page editor metabox |
| Short text on a specific page | ACF field on that page | Page editor sidebar |
| Global settings (social links, etc.) | Customizer | Appearance > Customize |
| Posts, media appearances, events | Custom post types | Their own sidebar menu items |

## Native WordPress first

In order of preference:

1. **Page content** (`the_content()`) — for text that belongs to a specific page
2. **Page title** (`the_title()`) — for the page heading
3. **Featured image** (`get_the_post_thumbnail_url()`) — for page-specific photography
4. **ACF fields on the page** — for structured data that doesn't fit the content editor
5. **Customizer** (`get_theme_mod()`) — for site-wide settings
6. **Custom post types** — for repeating structured content

## ACF fields — register in code

Always use `add_action('init', ..., 20)` — not `add_action('acf/init', ...)`. The `init` hook at priority 20 ensures `get_page_by_path()` and `get_option()` are available.

```php
add_action( 'init', function() {
    if ( ! function_exists( 'acf_add_local_field_group' ) ) return;

    acf_add_local_field_group( [
        'key'    => 'group_unique_key',
        'title'  => 'Field Group Label',
        'fields' => [
            [
                'key'   => 'field_unique_key',
                'label' => 'Field Label',
                'name'  => 'field_name',
                'type'  => 'text',
            ],
        ],
        'location' => [ [ [
            'param'    => 'post_type',
            'operator' => '==',
            'value'    => 'page',
        ] ] ],
    ] );
}, 20 );
```

## Custom post types

Use for any repeating structured content. Register in `functions.php`:

```php
register_post_type( 'my_event', [
    'labels' => [
        'name'         => 'Events',
        'singular_name'=> 'Event',
        'all_items'    => 'All Events',
        'add_new_item' => 'Add New Event',
        'menu_name'    => 'Events',
    ],
    'public'       => true,
    'show_ui'      => true,
    'show_in_rest' => true,
    'menu_icon'    => 'dashicons-calendar',
    'supports'     => [ 'title', 'editor', 'thumbnail' ],
] );
```

Always set the full label array — WordPress falls back to generic strings that make the admin confusing with multiple CPTs.

## Sanitization cheatsheet

| Situation | Function |
|-----------|----------|
| Plain text from `$_POST` | `sanitize_text_field( wp_unslash( $val ) )` |
| Multiline text from `$_POST` | `sanitize_textarea_field( wp_unslash( $val ) )` |
| Email address | `sanitize_email( wp_unslash( $val ) )` |
| URL | `esc_url_raw( wp_unslash( $val ) )` |
| Outputting text in HTML | `esc_html( $var )` |
| Outputting a URL in `href` | `esc_url( $var )` |
| Outputting in an HTML attribute | `esc_attr( $var )` |

Always use `wp_unslash()` before sanitizing user input — `sanitize_text_field()` applied directly to `$_POST` values will double-escape backslashes and mangle apostrophes.

---

# Performance Standards

## Images — the highest-impact optimization

Images are almost always the single biggest cause of poor PageSpeed scores.

- Serve at 2x the maximum display width (for retina). A 400px display width -> 800px image file.
- Compress JPEGs to quality 75-80.
- Add `width` and `height` attributes to every `<img>` tag.
- LCP element: `fetchpriority="high"`.
- Images below the fold: `loading="lazy"`.

**Hero images on page templates:** Preload via `wp_head`, not in the template body:

```php
add_action( 'wp_head', function() {
    if ( is_page( 'about' ) ) {
        $page = get_page_by_path( 'about' );
        $img  = $page ? get_the_post_thumbnail_url( $page->ID, 'large' ) : '';
        if ( $img ) {
            echo '<link rel="preload" as="image" href="' . esc_url( $img ) . '">' . "\n";
        }
    }
}, 2 );
```

A `<link rel="preload">` placed in the `<body>` is invalid HTML and browsers may ignore it.

## Google Fonts — non-render-blocking

Never load via CSS `@import`. Use the async `media="print"` pattern:

```php
$url = esc_url( $fonts_url );
?>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="preload" as="style" href="<?php echo $url; ?>">
<link rel="stylesheet" href="<?php echo $url; ?>" media="print" onload="this.media='all'">
<noscript><link rel="stylesheet" href="<?php echo $url; ?>"></noscript>
<?php
```

Use `display=swap` (not `display=optional`). `optional` causes inconsistent rendering across pages.

## Dark mode flash prevention

Add a tiny inline script to `<head>` that applies dark mode before any rendering:

```php
<script>(function(){var t=localStorage.getItem('theme'),d=window.matchMedia('(prefers-color-scheme:dark)').matches;if(t==='dark'||(t===null&&d))document.documentElement.setAttribute('data-theme','dark');})();</script>
```

Zero performance cost — ~150 bytes, no network request, under a millisecond.

## CSS line clamping and flex

`-webkit-line-clamp` requires `display: -webkit-box`. These elements collapse to zero height inside a flex container if you remove `flex: 1`. Never remove `flex: 1` from a clamped element — any spacing attempts on a zero-height element will silently fail.

## PageSpeed targets

| Metric | Target |
|--------|--------|
| Performance (mobile) | >= 85 |
| Accessibility | >= 95 |
| Best Practices | 100 |
| SEO | 100 |
| LCP (mobile) | < 4.0s |
| CLS | 0 |

---

# Accessibility Standards (WCAG 2.1 AA)

## Color contrast

- Body text: minimum 4.5:1 against background
- Large text (18pt+): minimum 3:1
- UI components: minimum 3:1

Always create a separate darker variable for text uses of the accent color.

## Keyboard navigation

- All interactive elements reachable by keyboard
- Visible focus indicator: `:focus-visible { outline: 2px solid var(--accent-text); outline-offset: 3px; }`
- Never use `:focus { outline: none }`

## Skip link

Every page must have a skip link as the first focusable element:
```html
<a class="skip-link" href="#main-content">Skip to content</a>
```

## Landmarks

- Use `<main id="main-content">` on every page
- Use `<nav aria-label="Primary">` for the main navigation
- When there are multiple `<nav>` elements, every one needs a distinct `aria-label`

## Modals and dialogs

- Use `aria-label` directly on the dialog element, not `aria-labelledby`. When the dialog starts hidden, `aria-labelledby` pointing to an element inside the hidden subtree resolves to nothing.
- Update `aria-label` via JS when the modal opens
- Close on: close button, backdrop click, Escape key
- Return focus to the trigger element when closed

## bfcache compatibility

Avoid `unload` event listeners (prevents bfcache in Firefox). Use `pageshow` instead of `load` for restored pages:

```js
window.addEventListener('pageshow', function(e) {
    if (e.persisted) {
        // page was restored from bfcache
    }
});
```
