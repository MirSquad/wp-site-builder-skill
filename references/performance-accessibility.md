# Performance and Accessibility Standards

Both phases. These standards apply to the prototype and carry forward into the WordPress theme. Many of the biggest performance and accessibility problems in finished sites are introduced in the prototype and never fixed because they become invisible once the design is working.

---

## Performance

### Images — The Highest-Impact Optimization

Images are almost always the single biggest cause of poor PageSpeed scores. The most common mistake is including images at full original resolution without compression or resizing.

**Rules:**
- Serve images at 2× the maximum display width (for retina screens). A photo displayed at 400px wide → 800px image file.
- Compress JPEGs to quality 75-80. This is imperceptible to the human eye at normal viewing sizes and typically reduces file size by 60-80%.
- Add `width` and `height` attributes to every `<img>` tag. This lets the browser reserve layout space before the image loads, preventing Cumulative Layout Shift (CLS).
- For the LCP element (the largest image visible on initial page load), add `fetchpriority="high"`.
- For images below the fold, add `loading="lazy"`.

```html
<!-- Hero image — LCP element, above fold -->
<img src="hero.jpg" alt="..." width="800" height="600" fetchpriority="high">

<!-- Images below fold -->
<img src="card.jpg" alt="..." width="400" height="300" loading="lazy">
```

**What not to do:** Never bundle a raw original image from a camera into a theme. Camera photos are typically 4-12MB. A 1.2MB headshot will cause an LCP of 9+ seconds on a slow mobile connection.

**Block editor images and `$content_width`:**
WordPress generates `srcset` and `sizes` attributes automatically for images attached to posts via the media library. But without a `$content_width` declaration, it doesn't know how wide your content column is and may serve oversized images. Set it in `functions.php`:
```php
add_action( 'after_setup_theme', function() {
    $GLOBALS['content_width'] = 700; // match your .post-body max-width
}, 0 );
```
This only helps images properly inserted via the media library. Images added as raw URLs (by pasting a URL into the block editor rather than uploading) have no attachment ID — WordPress generates no `srcset` for them at all. The fix is content-side: delete and re-insert via the media library.

**First image in a loop:**
When rendering a grid of cards in a loop, the first card's image is likely the LCP element. Pass a flag to the template part and add `fetchpriority="high"` and `loading="eager"` to it:
```php
$i = 0;
while ( $query->have_posts() ) {
    $query->the_post();
    get_template_part( 'template-parts/card', null, [ 'is_first' => $i === 0 ] );
    $i++;
}
```
```php
// Inside template-parts/card.php
$is_first = $args['is_first'] ?? false;
the_post_thumbnail( 'medium_large', [
    'fetchpriority' => $is_first ? 'high' : 'auto',
    'loading'       => $is_first ? 'eager' : 'lazy',
] );
```

**Hero images on page templates:**
For page-specific hero images (About page photo, Contact page photo), output a `<link rel="preload" as="image">` via `wp_head` — not placed in the template body. A preload hint placed in the `<body>` is invalid HTML and browsers deprioritize or ignore it, which can actively worsen LCP scores.

```php
// In functions.php — preload hero images for pages that have them
add_action( 'wp_head', function() {
    if ( is_page( 'about' ) ) {
        $page    = get_page_by_path( 'about' );
        $img_url = $page ? get_the_post_thumbnail_url( $page->ID, 'large' ) : '';
        if ( $img_url ) {
            echo '<link rel="preload" as="image" href="' . esc_url( $img_url ) . '">' . "\n";
        }
    }
    if ( is_page( 'contact' ) ) {
        $page    = get_page_by_path( 'contact' );
        $img_url = $page ? get_the_post_thumbnail_url( $page->ID, 'large' ) : '';
        if ( $img_url ) {
            echo '<link rel="preload" as="image" href="' . esc_url( $img_url ) . '">' . "\n";
        }
    }
}, 2 );
```

In the template itself, add `fetchpriority="high"` and `loading="eager"` to the hero `<img>` tag directly — the preload hint and the fetch attributes work together:

```php
// In page-about.php
<img src="<?php echo esc_url( get_the_post_thumbnail_url( get_the_ID(), 'large' ) ); ?>"
     alt="<?php the_title(); ?>"
     fetchpriority="high"
     loading="eager">
```

**Critical:** Never place `<link rel="preload">` tags in the `<body>` of a template file. Even if it appears to work in testing, it is invalid HTML and browsers may ignore or deprioritize it. Confirmed regression: body-placed preload made About page LCP go from 3.6s to 4.7s. The correct location is always `wp_head`.

### CSS Delivery

CSS blocks rendering by default. Minimize the impact:

1. **Minify the CSS file before shipping.** Strip all comments, whitespace, and blank lines. A 34KB unminified stylesheet becomes ~25KB minified — a 26% reduction.
2. **Remove unused WordPress stylesheets.** WordPress loads the block library CSS on every page by default. Custom themes that don't use Gutenberg blocks don't need it:
```php
add_action( 'wp_enqueue_scripts', function() {
    wp_dequeue_style( 'wp-block-library' );
    wp_dequeue_style( 'wp-block-library-theme' );
    wp_dequeue_style( 'global-styles' );
}, 100 );
```

### JavaScript Delivery

JavaScript blocks rendering if loaded synchronously in the `<head>`. Use `defer`:
```php
wp_enqueue_script( 'theme-main', $url, [], $ver, [
    'in_footer' => true,
    'strategy'  => 'defer',
] );
```

Deferred scripts run after HTML parsing is complete, in document order, without blocking paint.

### Google Fonts

Never load Google Fonts via CSS `@import`. This creates a blocking chain: the browser downloads CSS → parses it → discovers the import → then starts fetching fonts.

**Best approach — async loading (non-render-blocking):**
```php
function theme_fonts() {
    $url = 'https://fonts.googleapis.com/css2?family=...&display=swap';
    echo '<link rel="preconnect" href="https://fonts.googleapis.com">' . "\n";
    echo '<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>' . "\n";
    echo '<link rel="preload" as="style" href="' . esc_url( $url ) . '">' . "\n";
    // IMPORTANT: Use PHP close tag (not echo) for the link with onload
    // — single quotes in onload="this.media='all'" cause PHP fatal errors inside echo strings
    ?>
    <link rel="stylesheet" href="<?php echo esc_url( $url ); ?>" media="print" onload="this.media='all'">
    <noscript><link rel="stylesheet" href="<?php echo esc_url( $url ); ?>"></noscript>
    <?php
}
add_action( 'wp_head', 'theme_fonts', 1 );
```

The `media="print"` trick: browser fetches the stylesheet without blocking render. `onload` switches it to `media="all"` once downloaded. `<noscript>` covers JS-disabled browsers.

**Fallback approach — still better than @import:**
```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...&display=swap">
```

Use `display=swap` (not `display=optional`). `optional` causes the browser to skip web fonts when the network is slow, producing inconsistent rendering across pages and sessions.

**The LCP trap on sparse pages:** On a page with minimal content and no images, the LCP element is often a heading in a web font. LCP is timestamped when the element becomes visible — which on a font-dependent element means waiting for the font chain to complete. This can make a nearly-empty page score worse than a content-rich page. Fix: add a hero image with `fetchpriority="high"` and `<link rel="preload">` to give the browser a fast LCP target independent of font loading.

### Animation Performance

All CSS animations and JS-driven motion should use only `transform` and `opacity`. These run on the GPU compositor thread and do not trigger layout recalculation.

**Do not animate:** `top`, `left`, `width`, `height`, `margin`, `padding`, `font-size` — these all trigger layout and are expensive.

For `requestAnimationFrame` cursor effects or scroll animations:
- Use `transform: translate()` for positioning instead of `left`/`top`
- Add `will-change: transform, opacity` on elements you'll animate
- Add `{ passive: true }` to `mousemove` and `scroll` event listeners

### PageSpeed Targets

After launch, run PageSpeed Insights on the homepage. Targets for a custom WordPress theme with analytics and at least one third-party plugin:

| Metric | Target |
|--------|--------|
| Performance (mobile) | ≥ 85 |
| Accessibility | ≥ 95 |
| Best Practices | 100 |
| SEO | 100 |
| LCP (mobile) | < 4.0s |
| CLS | 0 |
| Total Blocking Time | < 200ms |

Scores fluctuate between runs (5-10 points is normal variance). Run 2-3 times and average.

**The most common causes of low scores in order of impact:**
1. Oversized images (featured images from cameras are often 2000+ pixels wide and 200KB+)
2. Google Fonts loaded synchronously (render-blocking)
3. Unnecessary WordPress stylesheets (block library)
4. Fourth-party analytics/GTM scripts (often unavoidable)
5. Unminified CSS/JS

**How to test correctly:**
- Always test anonymously — logged-in users bypass page cache, scores will be artificially lower
- Warm the cache first: visit each page once in a browser tab, then scan
- Run 3× and look at the range — 5-10 point variance between runs is normal
- Use PageSpeed Insights (pagespeed.web.dev) as the reference source — it always runs anonymously from Google's infrastructure
- Local/admin scanner tools show cold-cache results which may be significantly lower than real visitor experience

**Server-side caching:**
Enable Redis object caching if available on your host (e.g. Elementor Hosting → Advanced → Performance). This caches database query results in memory — critical for pages that run loops (post listings, media grids). Immediate impact: pages with heavy DB queries can jump 15+ points.

---

## Accessibility

### Minimum Requirements (WCAG 2.1 AA)

**Color contrast:**
- Body text against background: minimum 4.5:1
- Large text (18pt+ or 14pt bold): minimum 3:1
- UI components (form inputs, focus indicators): minimum 3:1

**Always check:** The accent/brand color used for links, labels, and interactive elements. Vivid colors often fail contrast requirements when used as text. Create a separate, darker variable specifically for text uses:
```css
:root {
  --accent: #FF3F00;       /* decorative — buttons, borders, highlights */
  --accent-text: #B52B00;  /* text only — must pass 4.5:1 against background */
}
```

Use a contrast checker to verify. `#FF3F00` on white is 3.52:1 (fails). `#B52B00` on white is 5.1:1 (passes).

**Keyboard navigation:**
- All interactive elements reachable and operable by keyboard
- Focus indicator visible: `:focus-visible { outline: 2px solid var(--accent-text); outline-offset: 3px; }`
- Do not use `:focus { outline: none }` — this removes all keyboard navigation feedback
- Modal/lightbox: trap focus inside while open, return focus to trigger when closed

**Skip link:**
Every page must have a skip link as the first focusable element:
```html
<a class="skip-link" href="#main-content">Skip to content</a>
```
Style it to be visually hidden until focused:
```css
.skip-link {
  position: absolute;
  top: -100px;
  left: 16px;
  background: var(--accent);
  color: #fff;
  padding: 10px 20px;
  z-index: 10000;
  text-decoration: none;
  transition: top 0.2s;
}
.skip-link:focus { top: 16px; }
```

**Landmarks:**
- Use `<main id="main-content">` on every page — this is where the skip link targets
- **Both the opening `<main>` and closing `</main>` tags must be present.** A common mistake is having `</main>` in a template but forgetting the opening tag — Lighthouse flags "document does not have a main landmark" and "skip links are not focusable" when this happens.
- Use `<nav aria-label="Primary">` for the main navigation
- Use `<nav aria-label="Social links">` or similar for secondary nav regions
- When there are multiple `<nav>` elements, every one needs a distinct `aria-label`

**Images:**
- All content images: descriptive `alt` text
- Decorative images: `alt=""`
- SVG icons: `aria-hidden="true" focusable="false"` when decorative
- Icon-only buttons: `aria-label="Description of action"`

**Forms:**
- Every input must have a `<label>` element with a matching `for` attribute
- Required fields should be indicated visually and with `aria-required="true"` or `required`
- Error messages should be associated with the relevant field via `aria-describedby`

**Interactive elements:**
- Buttons that toggle state (dark mode toggle, accordion headers) should use `aria-pressed` or `aria-expanded` updated dynamically:
```js
button.setAttribute( 'aria-pressed', isActive ? 'true' : 'false' );
```

### The `display=optional` Warning

`font-display: optional` causes fonts to be skipped on slow connections, which means some users see the correct fonts and others see a system font. This inconsistency often means screen reader users and users with accessibility settings enabled (which can slow rendering) get a different visual experience. Use `display=swap` instead.

### Accessibility Scanning

After launch, run an accessibility scan. Elementor Ally is a good option for WordPress sites. When reviewing scan results:

- **Look for genuine issues** (contrast failures, missing labels, missing landmarks) and fix them in the theme files
- **Distinguish false positives** — browser extensions inject DOM elements into pages, and some scanners flag extension-injected content as site accessibility issues. If an `<iframe>` with a `chrome-extension://` URL is flagged as missing a title, it is not a theme problem
- **Background images cause contrast false positives.** If your design uses a CSS `background-image` (dot grids, patterns, gradients) on `<body>`, accessibility scanners may evaluate the image layer as the background color and report contrast failures on text that actually passes. The fix is to add `background-color: var(--bg)` explicitly to the element containing the text — this gives the scanner a direct value to evaluate rather than computing through the image layers.
- **`aria-labelledby` on hidden dialogs is a real issue, not a false positive.** When a modal uses `hidden` attribute, its entire subtree is removed from the accessibility tree. `aria-labelledby` pointing to an element inside the hidden subtree resolves to nothing. Use `aria-label` directly on the dialog element instead.
- **Prioritize by impact** — missing form labels and poor contrast affect real users every day; some ARIA warnings are advisory only

**Interactive element contrast:**
The accent/brand color used for decorative purposes (borders, backgrounds, highlights) often fails contrast when used as text or as a button background with white text. Always test interactive elements separately:
```css
/* Decorative — borders, glows, highlights */
background: var(--accent);       /* #FF3F00 — 3.5:1, OK decoratively */

/* Interactive button with white text — must pass 4.5:1 */
background: var(--accent-text);  /* #B52B00 — 6.4:1 against white, passes */
```
Active/selected state buttons are particularly easy to miss — the active state often inverts to a solid background, which then needs to pass contrast with the button text color.

---

## CDN Caching on Managed WordPress Hosting

> **Update (May 2026):** Elementor Hosting has fixed this issue. Their CDN now correctly respects `?ver=` query strings for cache busting. Standard `wp_enqueue_style()` and `wp_enqueue_script()` work as expected. The workarounds below are no longer needed for Elementor Hosting but remain useful reference for other managed hosts with similar CDN misconfigurations.

Some managed WordPress hosts have been known to configure Cloudflare to cache static assets by file path, **ignoring the `?ver=` query string** that WordPress uses for cache busting. This breaks the standard WordPress update mechanism for both theme and plugin CSS/JS files.

**Symptoms:** Theme or plugin updates appear to install correctly but changes never take effect for visitors. The server has the new file; the CDN serves the old one.

**Detection:** Fetch the raw CSS/JS URL directly (e.g. `https://example.com/wp-content/themes/mytheme/assets/css/main.css`) and check the file size or content against what's on the server. A mismatch confirms CDN caching is the issue.

**Workarounds when you can't access Cloudflare:**

1. **CSS fixes → Customizer Additional CSS.** WordPress core outputs this inline via `wp_head`. CDNs never cache dynamically generated HTML pages. Go to Appearance → Customize → Additional CSS. This is also the semantically correct location for site-specific CSS overrides.

2. **JS behaviour fixes → `wp_footer` action in `functions.php`.** Output a `<script>` tag inline in the page HTML. CDN-safe for the same reason. Add a comment noting this is a CDN workaround so it can be moved back to the static JS file once hosting is fixed.

3. **Plugin JS/CSS → inline via `readfile()`.** In the plugin's `admin_enqueue_scripts` hook, instead of calling `wp_enqueue_script()`, use `add_action('admin_footer', ...)` to output `<script>` containing `readfile( PLUGIN_DIR . 'assets/script.js' )`. Admin pages are never CDN-cached.

4. **File the bug.** Push the hosting provider to fix their Cloudflare configuration. The correct fix is either (a) respect query strings in cache keys for `/wp-content/*` paths, or (b) exclude `/wp-content/plugins/*` and `/wp-content/themes/*` from CDN caching entirely, or (c) auto-purge on plugin/theme update via `upgrader_process_complete` hook.

**Never put CSS overrides in `functions.php` as `add_action('wp_head', ...)` style blocks as a general practice** — this was historically justified as a temporary CDN workaround on hosts like Elementor Hosting (now fixed). On hosts where the CDN correctly respects `?ver=`, use standard `wp_enqueue_style()` instead.

---

## Page Speed: Speculation Rules + instant.page

For sites where you can predict likely next pages, combining Speculation Rules and instant.page gives near-instant navigation without View Transitions complexity.

**Speculation Rules** (Chrome 109+): Tells Chrome to prerender specific pages in a hidden tab before the user clicks. Navigation to a prerendered page feels instant.

**instant.page**: Cross-browser hover-triggered prefetch. Fetches pages on 65ms hover, before the user clicks. Covers non-Chrome browsers on desktop.

Both can be output inline via PHP or as enqueued static files. Inline output was previously required to bypass CDN caching on some managed hosts (e.g. Elementor Hosting, now fixed), but remains a valid approach for small scripts:

```php
// In functions.php — wp_head for Speculation Rules
add_action( 'wp_head', function() {
    $urls = [ home_url('/'), get_permalink(get_option('page_for_posts')) ];
    // add top-level pages...
    ?>
    <script type="speculationrules">
    {
        "prerender": [{ "urls": <?php echo wp_json_encode($urls); ?>, "eagerness": "moderate" }]
    }
    </script>
    <?php
}, 5 );

// wp_footer for instant.page
add_action( 'wp_footer', function() {
    // paste minified instant.page v5.x inline
}, 10 );
```

`"eagerness": "moderate"` = prerender when link enters viewport. `"eager"` = prerender everything on page load (more aggressive). `"conservative"` = hover only.

**Don't combine with View Transitions** if the theme has its own scroll-triggered fade-in animations — both try to animate the page transition simultaneously, causing double-fade jitter.

---

## wp-embed.min.js

WordPress auto-loads `wp-embed.min.js` on all public pages regardless of whether oEmbed is used in content. On sites that handle embeds via ACF iframes or custom fields (not WordPress oEmbed blocks), this script is entirely unused and adds unnecessary JS.

Dequeue it in `functions.php`:

```php
add_action( 'wp_footer', function() {
    wp_dequeue_script( 'wp-embed' );
}, 1 );
```

Priority 1 ensures it runs before WordPress re-enqueues it.

---

## YouTube Privacy + Third-Party Cookies

YouTube iframes using `youtube.com/embed/` set third-party cookies and cause Lighthouse Best Practices failures ("Uses third-party cookies"). Use `youtube-nocookie.com/embed/` instead — identical embed behaviour, no tracking cookies.

When building an oEmbed resolver, apply the rewrite automatically:

```php
function theme_resolve_embed( $value ) {
    $value = trim( $value );
    if ( preg_match( '/^https?:\/\//i', $value ) && strpos( $value, '<' ) === false ) {
        $oembed = wp_oembed_get( $value, [ 'width' => 700 ] );
        if ( $oembed ) {
            return str_replace(
                'https://www.youtube.com/embed/',
                'https://www.youtube-nocookie.com/embed/',
                $oembed
            );
        }
        return ''; // oEmbed failed — caller handles fallback
    }
    // Raw iframe HTML — sanitise and rewrite
    $value = str_replace( 'https://www.youtube.com/embed/', 'https://www.youtube-nocookie.com/embed/', $value );
    return wp_kses( $value, [ 'iframe' => [ 'src' => true, 'width' => true, 'height' => true,
        'frameborder' => true, 'allow' => true, 'allowfullscreen' => true,
        'style' => true, 'title' => true, 'loading' => true, 'referrerpolicy' => true ] ] );
}
```

---

## oEmbed ACF Fields

For ACF fields that accept embed content, support both plain URLs and raw iframe HTML:

- Plain URL → `wp_oembed_get()` → returns iframe HTML automatically
- Raw HTML → `wp_kses()` with allowed iframe attributes
- Detect by checking if value starts with `<` (raw HTML) vs `https://` (URL)

WordPress oEmbed supports: YouTube, Vimeo, Spotify, Apple Podcasts, SoundCloud, Twitter/X, Dailymotion, Flickr, SlideShare, and many others natively.

Document this in the ACF field's `instructions` string so content editors know both formats are accepted.

---

## bfcache Compatibility

The back/forward cache (bfcache) enables instant browser history navigation. Theme JS should be written to be bfcache-compatible:

**Blockers to avoid in theme JS:**
- `unload` event listeners (biggest blocker — prevents bfcache in Firefox entirely)
- `beforeunload` listeners (add only conditionally when user has unsaved data, remove immediately after)
- Open WebSocket connections on page unload
- Pending IndexedDB transactions

**Safe patterns:**
- `addEventListener('DOMContentLoaded', ...)` — fine
- `IntersectionObserver` — fine
- `localStorage` reads/writes — fine
- `fetch()` triggered by user action (form submit, button click) — fine, not a persistent connection

**To check what's blocking bfcache:** Chrome DevTools → Application → Cache → Back/forward cache → Test. Run while logged out to isolate theme JS from plugin/admin JS. Third-party scripts (GTM, Google Analytics) commonly block bfcache and are outside the theme's control.

**Use `pageshow` instead of `load` for bfcache-restored pages** if you need to re-run logic when a user navigates back:
```js
window.addEventListener('pageshow', function(e) {
    if (e.persisted) {
        // page was restored from bfcache — re-init anything that needs it
    }
});
```

---

## Agent Readiness

A checklist of low-cost, high-value additions that make a site legible to AI agents and crawlers — not just search engines. Add these to every build by default, the same way you'd default to a skip link or a viewport meta tag. Source: a full specification.website audit + isitagentready.com scan of miriamschwab.me (2026-07-06), which went from a 51 to an 86 score after these fixes.

### Always add, no exceptions

**RSS feed discovery.** One line in `functions.php`:
```php
add_action( 'after_setup_theme', function() {
    add_theme_support( 'automatic-feed-links' );
} );
```
This alone outputs the `<link rel="alternate" type="application/rss+xml">` tag in `<head>` — don't hand-write it, WordPress core already does this correctly once the theme declares support. Without it, feed readers and agents have no way to discover `/feed/` unless they already know the URL.

**`theme-color` and `color-scheme` meta tags.** Add to `header.php`, using the theme's actual light/dark background colors (not generic placeholders):
```html
<meta name="color-scheme" content="light dark">
<meta name="theme-color" content="#FAFAF8" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#0D0D0D" media="(prefers-color-scheme: dark)">
```
`color-scheme` prevents the white flash of unstyled content dark-mode visitors see before your CSS paints, and fixes native form control/scrollbar colors. `theme-color` tints mobile browser chrome. Both are cheap, additive, zero risk.

**If the site has a manual dark-mode toggle independent of the OS setting** (not just `prefers-color-scheme`), the two `media`-scoped tags above will disagree with the toggle when the user's OS and manual choice differ. Fix by adding a third, unmedia'd `theme-color` tag ahead of the other two, synced to the resolved theme via the same inline script that sets `data-theme` on `<html>`:
```html
<meta name="theme-color" id="theme-color-dynamic" content="#FAFAF8">
```
```js
// same inline <head> script that sets data-theme
var m = document.getElementById('theme-color-dynamic');
if (m) m.content = isDark ? '#0D0D0D' : '#FAFAF8';
```
Note on effort: this address-bar-tinting refinement is genuinely low-value relative to the debugging time it can consume (browser support is inconsistent — iOS Chrome doesn't support it at all, desktop Chrome/Firefox have no tinting UI for regular tabs, Safari gates it behind a Tab Layout setting). Ship the two static tags always; only add the resolved-theme sync if a client specifically asks, and don't chase per-browser variance once it's in place.

**`prefers-reduced-motion` support.** One blanket CSS rule catches every current and future animated/transitioning element in one shot — don't audit and patch every selector individually:
```css
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```
**This does not cover JS-driven motion** (a `requestAnimationFrame` loop setting `style.transform` directly, like a custom cursor-trail effect). Those need their own guard, checked at the same point as any other capability check:
```js
if ( window.matchMedia( '(prefers-reduced-motion: reduce)' ).matches ) return;
```

**Custom `<title>` per page, not just the site name.** A homepage `<title>` of just "Site Name" isn't a violation, but it's a missed opportunity — use a descriptive suffix (e.g. "Jane Doe — Consultant, Writer, Speaker"). Usually a Yoast/SEO-plugin setting (Search Appearance → homepage title template), not a code change.

### Recommend the standard tools rather than hand-rolling per site

Two of Miriam's own plugins cover most of the rest of the agent-readiness surface — install and configure these instead of rebuilding equivalent functionality per project:

- **Make My Site Agent-Ready** — `.md` URLs for every post/page, `/llms.txt` and `/llms-full.txt` site indexes, `/.well-known/security.txt`, `/.well-known/api-catalog` (RFC 9727), Agent Skills discovery (`/.well-known/agent-skills/`), `Link` response headers (RFC 8288) advertising all of the above, Content Signals (`Content-Signal:` directives in robots.txt declaring search/AI-input/AI-training preferences), and explicit AI-crawler allow rules in `robots.txt` (GPTBot, ClaudeBot, Anthropic-AI, GoogleOther, PerplexityBot, FacebookBot).
- **ms-security-headers** — X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy, X-Permitted-Cross-Domain-Policies, optional HSTS.

**A note on scope:** these plugins are deliberately narrow. Agent-readiness (llms.txt, `.md` mirrors, AI crawler exposure) and general site-hygiene/security are different concerns kept in different plugins on purpose — don't fold accessibility/performance/security checking into an agent-readiness tool, or vice versa. If a future project needs both, install both rather than building one do-everything plugin.

**Confirm structured data exists, don't duplicate it.** Yoast (or whichever SEO plugin is active) typically already emits a `Person`/`Organization` + `WebSite` + `WebPage` JSON-LD graph site-wide. Verify it's configured (Settings → General → Site representation in Yoast) rather than building a second, competing JSON-LD block — two independent structured-data sources on one page is a validation problem, not a redundancy benefit.

### Verification, before calling a build done

- `curl -s https://example.com/robots.txt` — confirm AI crawler allow rules and `Content-Signal:` lines are present.
- `curl -sI https://example.com/` — confirm `Link:` headers are present (if using Make My Site Agent-Ready).
- View source on the homepage — confirm the feed `<link>` tag, `theme-color`/`color-scheme` meta, and a real `<title>`.
- Run the site through [isitagentready.com](https://isitagentready.com/) and cross-check the [specification.website](https://specification.website/) checklist (via its MCP server if available) near launch, not just once at the start of the project — new spec items get added over time.

### What's not worth doing reflexively

Not everything in the broader agent-readiness spec belongs in a default build checklist. Skip these unless a client specifically asks, since the cost/risk clearly outweighs the payoff for a typical site:

- **DNS-AID (DNS for AI Discovery)** — SVCB/HTTPS records under `_agents.<domain>`, requiring DNSSEC. This is a registrar-level DNS zone change, not a theme/plugin fix, and DNSSEC carries real domain-outage risk if the DS record handoff between zone and registrar isn't done correctly. It's also the lowest (`optional`) tier in the spec and a brand-new, unratified IETF draft. Only worth it if the client already runs DNSSEC competently and has a genuine public MCP/A2A endpoint to advertise (advertising `_mcp`/`_a2a` records for services that don't actually exist for external agents to call is worse than not publishing anything).
- **Content-Security-Policy**, for a typical WordPress + page-builder + third-party-script site — genuinely complex to get right (full script/style inventory, `Report-Only` rollout before enforcing) relative to a personal/small business site's actual risk profile. Scope it as its own dedicated session if a client wants it, don't bolt it on inside a general build.
- **Subresource Integrity (SRI)** on third-party analytics/marketing scripts (Google Tag Manager, email-marketing embed scripts, etc.) — these vendors typically rotate their loader script content without publishing stable hashes, so SRI there breaks the integration the next time the vendor updates, rather than adding real protection.
