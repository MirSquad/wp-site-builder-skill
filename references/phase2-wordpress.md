# Phase 2: WordPress Theme Architecture

## Theme File Structure

```
theme-name/
├── style.css               — Theme declaration (required)
├── functions.php           — All PHP: setup, CPTs, ACF, enqueue, helpers
├── header.php              — <html>, <head>, wp_head(), nav
├── footer.php              — footer markup, wp_footer()
├── index.php               — Fallback template (required)
├── front-page.php          — Homepage (used when static front page is set)
├── page.php                — Generic page fallback
├── page-about.php          — About page (applied automatically by filename)
├── page-contact.php        — Contact page
├── page-[slug].php         — Any page with matching slug
├── single.php              — Single blog post
├── archive.php             — Post archive / blog listing
├── 404.php                 — Not found page
├── single-[post-type].php  — Single CPT item (e.g. single-ms_media.php)
├── template-parts/
│   └── media-card.php      — Reusable partial templates
└── assets/
    ├── css/main.css         — Compiled from prototype
    ├── js/main.js           — Compiled from prototype
    └── images/              — Fallback images bundled with theme
```

## style.css — Theme Declaration

```css
/*
Theme Name: My Theme
Description: Custom theme for example.com
Version: 1.0.0
Author: Author Name
*/
```

No actual CSS goes here. All CSS goes in `assets/css/main.css`, enqueued via `functions.php`.

## functions.php Structure

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

// Declare content column width so WordPress picks the right srcset candidate
// for block editor images. Set to the max-width of your .post-body column.
function theme_content_width() {
    $GLOBALS['content_width'] = 700; // adjust to match your post body column width
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
   CUSTOMIZER
   ============================================= */
/* =============================================
   CONTACT FORM / AJAX (if needed)
   ============================================= */
/* =============================================
   HELPER FUNCTIONS
   ============================================= */
```

## Enqueuing Assets

```php
function theme_enqueue_assets() {
    // Always read version dynamically — never hardcode as a string.
    // The version from style.css is appended as ?ver=x.x.x to CSS/JS URLs.
    // If hardcoded, the browser caches the old file forever regardless of updates.
    // See "Version Management" section below.
    $ver = wp_get_theme()->get( 'Version' );

    // Google Fonts — do NOT enqueue via wp_enqueue_style.
    // Load async via the media="print" pattern in wp_head to avoid render-blocking.
    // See performance-accessibility.md → Google Fonts for the correct pattern.

    wp_enqueue_style( 'theme-main',
        get_template_directory_uri() . '/assets/css/main.css',
        [], $ver );

    wp_enqueue_script( 'theme-main',
        get_template_directory_uri() . '/assets/js/main.js',
        [], $ver,
        [ 'in_footer' => true, 'strategy' => 'defer' ]  // defer — don't block rendering
    );
}
add_action( 'wp_enqueue_scripts', 'theme_enqueue_assets' );

// Dequeue block library CSS — not needed for custom themes
add_action( 'wp_enqueue_scripts', function() {
    wp_dequeue_style( 'wp-block-library' );
    wp_dequeue_style( 'wp-block-library-theme' );
    wp_dequeue_style( 'global-styles' );
}, 100 );
```

The `strategy: defer` argument requires WordPress 6.3+. For older installs, use `true` as the fifth argument to place the script in the footer instead.

## header.php Structure

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

<div class="page-wrapper">
```

**`wp_head()` must be present** — plugins including SEO tools, analytics, and accessibility scanners inject critical code here.

**`wp_body_open()` must be present** — Google Tag Manager and other tools inject code here.

**The skip link must be the first focusable element in `<body>`**, before the nav.

**`<nav>` must have `aria-label`** whenever there's more than one nav region on the page.

## footer.php Structure

```php
</div><!-- .page-wrapper -->

<footer class="site-footer">
  <!-- footer content -->
</footer>

<?php wp_footer(); ?>
</body>
</html>
```

`wp_footer()` is required. Scripts enqueued for the footer are output here.

## Template Pages — The key pattern

For pages with custom templates (About, Contact, Media, etc.), always call `the_post()` at the top so native WP functions work correctly:

```php
<?php
/**
 * Template Name: About
 */
get_header();
the_post();
?>
<main id="main-content">
  <h1><?php the_title(); ?></h1>
  <div><?php the_content(); ?></div>
  <?php if ( has_post_thumbnail() ) : ?>
  <img src="<?php echo esc_url( get_the_post_thumbnail_url( get_the_ID(), 'large' ) ); ?>" alt="<?php the_title(); ?>">
  <?php endif; ?>
</main>
<?php get_footer(); ?>
```

Note: **`<main id="main-content">`** must be present on every template. This is where the skip link points.

## Contact Forms — Native wp_mail()

Do not install a form plugin (WPForms, Contact Form 7) unless the client specifically needs one. A basic contact form is straightforward to build natively:

```php
function theme_handle_contact() {
    if ( ! wp_verify_nonce( $_POST['nonce'] ?? '', 'contact_nonce' ) ) {
        wp_send_json_error( 'Security check failed.' );
    }
    $name    = sanitize_text_field( wp_unslash( $_POST['name']    ?? '' ) );
    $email   = sanitize_email(      wp_unslash( $_POST['email']   ?? '' ) );
    $message = sanitize_textarea_field( wp_unslash( $_POST['message'] ?? '' ) );

    if ( ! $name || ! $email || ! $message || ! is_email( $email ) ) {
        wp_send_json_error( 'Please fill in all required fields.' );
    }

    $to      = get_option( 'admin_email' );
    $subject = "New message from " . get_bloginfo('name');
    $body    = "Name: {$name}\nEmail: {$email}\n\nMessage:\n{$message}";
    $headers = [ 'Content-Type: text/plain; charset=UTF-8', "Reply-To: {$name} <{$email}>" ];

    wp_mail( $to, $subject, $body, $headers ) ?
        wp_send_json_success() :
        wp_send_json_error( 'Could not send message. Please try again.' );
}
add_action( 'wp_ajax_theme_contact',        'theme_handle_contact' );
add_action( 'wp_ajax_nopriv_theme_contact', 'theme_handle_contact' );
```

**Important:** `wp_mail()` requires the server to be able to send email, which many hosting environments block by default. Note this to the client and recommend an SMTP plugin (or the host's native SMTP solution) if emails are not arriving.

**Important:** Always use `wp_unslash()` before sanitizing user input. `sanitize_text_field()` and `sanitize_textarea_field()` applied directly to `$_POST` values will double-escape backslashes and mangle apostrophes in contractions.

```php
// CORRECT
$text = sanitize_text_field( wp_unslash( $_POST['field'] ?? '' ) );

// WRONG — mangles "I've" into "I\'ve" on each save
$text = sanitize_text_field( $_POST['field'] ?? '' );
```

For saved options/fields, use `wp_kses_post( wp_unslash( $value ) )` if the text might contain allowed HTML, or `sanitize_text_field( wp_unslash( $value ) )` for plain text.

## Post Body Image and Caption Styles

WordPress wraps captioned images in `<figure>` with a `<figcaption class="wp-element-caption">`. Without explicit theme styles, captions inherit body text size and color and look unstyled. Always include these in the post body CSS:

```css
.post-body figure { margin: 36px 0; }
.post-body figure img { margin: 0; } /* reset img margin — figure handles spacing */
.post-body figcaption,
.post-body .wp-element-caption {
    font-size: 12px;
    color: var(--text-dim);
    margin-top: 8px;
    line-height: 1.6;
}
.post-body figure.aligncenter { text-align: center; }
.post-body figure.alignleft   { float: left;  margin-right: 28px; }
.post-body figure.alignright  { float: right; margin-left:  28px; }
```

Both `figcaption` and `.wp-element-caption` must be targeted — older WordPress versions use `figcaption`, newer Gutenberg versions use the class.

## Lightboxes and UI Components

Build UI components (lightboxes, modals, accordions, tabs, carousels) in vanilla JS with no dependencies. Do not install jQuery plugins or JS libraries for individual components. A self-contained implementation:
- Has no version conflicts
- Adds no HTTP requests
- Can be fully styled to match the design
- Remains under the theme's control

For a lightbox/modal:
- Use `position: fixed` overlay with `z-index` above all other content
- Include `aria-modal="true"` and `role="dialog"` on the modal element
- **Use `aria-label` directly on the modal element, not `aria-labelledby`.** When the modal starts hidden (via `hidden` attribute or `display:none`), the entire subtree is excluded from the accessibility tree — `aria-labelledby` points to an element that doesn't exist in the tree and resolves to nothing. `aria-label` on the element itself works correctly even when hidden.
- Update `aria-label` via JS when the modal opens with the actual content title
- Close on: close button click, backdrop click, Escape key
- Clear iframe `src` on close to stop media playback
- Return focus to the trigger element when closed

```html
<!-- CORRECT: aria-label on the element itself -->
<div role="dialog" aria-modal="true" aria-label="Media player" hidden>

<!-- WRONG: aria-labelledby when the target is inside the hidden subtree -->
<div role="dialog" aria-modal="true" aria-labelledby="modal-title" hidden>
  <span id="modal-title"></span>  <!-- hidden — not in accessibility tree -->
```

```js
function openModal( content, title ) {
    modal.setAttribute( 'aria-label', title || 'Media player' );
    modal.removeAttribute( 'hidden' );
    // ...
}
```

## Menus

Register nav menus in `functions.php`:
```php
register_nav_menus( [ 'primary' => 'Primary Navigation' ] );
```

Output in `header.php`:
```php
wp_nav_menu( [
    'theme_location' => 'primary',
    'container'      => false,
    'menu_class'     => 'nav-menu',
    'fallback_cb'    => 'theme_fallback_nav',
] );
```

Always provide a `fallback_cb` so the nav doesn't disappear if no menu is assigned in the admin.

To add `./` prefixes or other decorations to nav item text without affecting accessibility, use the `nav_menu_item_title` filter:
```php
add_filter( 'nav_menu_item_title', function( $title, $item, $args, $depth ) {
    if ( $args->theme_location === 'primary' ) {
        return '<span class="nav-label">' . $title . '</span>';
    }
    return $title;
}, 10, 4 );
```

## Sanitization Cheatsheet

| Situation | Function |
|-----------|----------|
| Plain text from `$_POST` | `sanitize_text_field( wp_unslash( $val ) )` |
| Text with apostrophes saved to options | `wp_kses_post( wp_unslash( $val ) )` |
| Multiline text from `$_POST` | `sanitize_textarea_field( wp_unslash( $val ) )` |
| Email address | `sanitize_email( wp_unslash( $val ) )` |
| URL | `esc_url_raw( wp_unslash( $val ) )` |
| Outputting text in HTML | `esc_html( $var )` |
| Outputting HTML (from trusted source) | `wp_kses_post( $var )` |
| Outputting a URL in `href` or `src` | `esc_url( $var )` |
| Outputting in an HTML attribute | `esc_attr( $var )` |

## Version Management

Set the version in `style.css`. In `functions.php`, always read it dynamically — never hardcode it as a string:

```php
// CORRECT — reads from style.css, stays in sync automatically
$ver = wp_get_theme()->get( 'Version' );
wp_enqueue_style( 'theme-main', get_template_directory_uri() . '/assets/css/main.css', [], $ver );
wp_enqueue_script( 'theme-main', get_template_directory_uri() . '/assets/js/main.js', [], $ver, [ 'in_footer' => true, 'strategy' => 'defer' ] );

// WRONG — hardcoded version never changes, browser caches old CSS/JS forever
$ver = '1.0.0';
```

**Why this matters:** The version string is appended as a query parameter to CSS and JS URLs (`main.css?ver=1.0.0`). Browsers and CDNs use this to cache assets. If `$ver` is hardcoded, installing a new theme zip with updated CSS has no effect — the browser keeps serving the old cached file because the URL hasn't changed. With dynamic version reading, bumping the version in `style.css` automatically busts the cache.

**Increment the version in `style.css` with every delivery.** Use patch versioning: `1.0.0` → `1.0.1` → `1.0.2`. This also lets you verify in Appearance → Themes → Theme Details that the correct version is installed — the fastest way to confirm an update took effect.

---

## Mobile Navigation Patterns

When a nav bar has a horizontally scrollable menu on mobile, any UI controls that must always be visible (dark mode toggle, search, etc.) must be placed **outside** the scrollable container — as siblings, not children.

```php
// header.php — CORRECT: toggle is sibling of scrollable nav
<div class="container">
    <a class="nav-logo" href="...">Logo</a>
    <div class="nav-right">  <!-- scrollable on mobile -->
        <?php wp_nav_menu([...]); ?>
    </div>
    <button class="theme-toggle">...</button>  <!-- always visible -->
</div>
```

```css
/* mobile CSS */
@media (max-width: 640px) {
    .site-nav .container { flex-direction: row; flex-wrap: wrap; align-items: center; }
    .nav-right { order: 3; width: 100%; overflow-x: auto; }
    .nav-logo { flex: 1; }
    .theme-toggle { order: 2; }
}
```

Also add `white-space: nowrap` to nav links so multi-word labels ("Media & Talks") don't wrap within the link element itself:
```css
.nav-menu a { white-space: nowrap; }
```

---

## Embed Resolution Helper

For themes with ACF embed fields, centralise resolution logic in a single helper rather than duplicating `wp_kses()` calls across templates:

```php
function theme_resolve_embed( $value ) {
    if ( empty( $value ) ) return '';
    $value = trim( $value );

    // Plain URL — try oEmbed
    if ( preg_match( '/^https?:\/\//i', $value ) && strpos( $value, '<' ) === false ) {
        $result = wp_oembed_get( $value, [ 'width' => 700 ] );
        return $result ?: ''; // empty string = caller shows fallback link
    }

    // Raw iframe HTML — sanitise
    return wp_kses( $value, [
        'iframe' => [
            'src' => true, 'width' => true, 'height' => true,
            'frameborder' => true, 'allow' => true, 'allowfullscreen' => true,
            'style' => true, 'title' => true, 'loading' => true,
            'scrolling' => true, 'referrerpolicy' => true,
        ],
    ] );
}
```

In templates, check the resolved value for emptiness and show an appropriate fallback (external link button) when oEmbed fails:

```php
$resolved = theme_resolve_embed( get_field('embed') );
if ( $resolved ) {
    echo $resolved;
} elseif ( get_field('embed') ) {
    // oEmbed failed but a URL was provided — show external link
    echo '<a href="' . esc_url( get_field('embed') ) . '" target="_blank">View on external site</a>';
}
```

---

## CSS Line Clamping and Flex Spacing

`-webkit-line-clamp` is the standard way to truncate text to a fixed number of lines. It requires `display: -webkit-box` and `-webkit-box-orient: vertical`. This has a non-obvious side effect that causes hours of debugging if you don't know about it:

**`-webkit-box` elements collapse to zero height inside a flex container when `flex-grow` is removed.**

If you apply `flex: none` or remove `flex: 1` from a clamped element, its computed height becomes zero — even though text visually overflows the box. Any `margin-bottom` on a zero-height element has no physical space to occupy and is invisible regardless of value. The same applies to `padding-top` on the following sibling. You can iterate on CSS values all day and see no change.

```css
/* The clamping setup — requires these three together */
.card-desc {
    display: -webkit-box;
    -webkit-box-orient: vertical;
    -webkit-line-clamp: 3;
    overflow: hidden;
    flex: 1; /* DO NOT remove this — removing it collapses height to zero */
}
```

**How to detect this:** If spacing between a clamped element and its sibling looks wrong, measure actual height before iterating on values:
```js
document.querySelector('.card-desc').getBoundingClientRect().height
// If this returns 0, the element has no physical space — CSS spacing fixes are irrelevant
```

**The fix when you need guaranteed spacing below a clamped element:** Use a DOM spacer element between the clamped text and the element that should sit below it. A real DOM element has genuine height that the browser respects:

```php
<!-- In template-parts/card.php -->
<p class="card-desc"><?php the_excerpt(); ?></p>

<!-- DOM spacer: pushes footer to bottom on short content, collapses gracefully on long content -->
<div style="flex:1;min-height:20px"></div>

<footer class="card-meta">...</footer>
```

This also bypasses CDN caching problems: the spacer lives in a PHP template file, which is always served dynamically. A CSS-only fix in a static asset file may not reach visitors until the CDN cache expires.

**Never remove `flex: 1` from a `-webkit-box` clamped element** to try to fix spacing. It will collapse the element height and any spacing attempts will silently fail.

---

## Dead Code Patterns to Avoid

- **`register_activation_hook()` in theme files** — this hook only works in plugins. Use `add_action('after_switch_theme', ...)` for theme activation logic (flush rewrite rules, seed options, etc.). Note: fires on theme switch, not on zip update.
- **Unused helper functions** — periodically grep all function definitions against the codebase to find functions that are defined but never called. Remove them.
- **Empty comment sections** — placeholder comments left from scaffolding with no code beneath them add noise with no value.
- **Hardcoded version strings in `functions.php`** — always use `wp_get_theme()->get('Version')` so the cache buster stays in sync with `style.css` automatically.


---

## Dark Mode — Preventing Flash on Page Load

If dark mode is stored in `localStorage` and applied via JavaScript, the browser will paint the page in light mode first, then apply dark mode when the script runs. This causes a visible flash of light mode on every page navigation — especially noticeable when `main.js` is deferred.

**The fix:** Add a tiny inline script to `<head>`, before the closing `</head>` tag, that applies dark mode before any rendering occurs:

```php
// In header.php, just before </head>
?>
<script>/* Dark mode — inline in <head> to prevent flash */
(function(){var t=localStorage.getItem('ms-theme'),d=window.matchMedia('(prefers-color-scheme:dark)').matches;if(t==='dark'||(t===null&&d))document.documentElement.setAttribute('data-theme','dark');})();
</script>
</head>
```

Inline scripts in `<head>` run synchronously before any rendering. This means `data-theme="dark"` is set on `<html>` before CSS is applied — no flash.

**What to keep in the deferred JS file:** The toggle button click handler, the `aria-pressed` updates, and the `localStorage.setItem()` call. Only the init (read localStorage + set attribute) needs to be inline.

**Zero performance cost:** ~150 bytes, no network request, runs in under a millisecond.

---

## PHP Quoting Trap with HTML `onload` Attributes

When outputting HTML that contains `onload="this.media='all'"` (used for async Google Fonts loading), the single quotes inside the attribute cause PHP fatal errors if output via `echo`:

```php
// FATAL — single quotes break out of the PHP string
echo '<link rel="stylesheet" media="print" onload="this.media=\'all\'">';
```

Even with escaping, this is fragile. The correct approach is to drop out of PHP mode entirely:

```php
$url = esc_url( $fonts_url );
?>
<link rel="stylesheet" href="<?php echo $url; ?>" media="print" onload="this.media='all'">
<noscript><link rel="stylesheet" href="<?php echo $url; ?>"></noscript>
<?php
```

In PHP-close mode, the single quotes inside `onload` are literal HTML characters — no escaping needed.

---

## Custom Post Type Labels — Always Set the Full Label Array

WordPress falls back to generic strings ("All Items", "Add New Item", "Edit Item") for any label not explicitly set. When you have multiple CPTs this makes the admin sidebar confusing — every CPT shows identical submenu text.

Always set specific labels when registering a CPT:

```php
register_post_type( 'ms_media', [
    'labels' => [
        'name'          => 'Press & Talks',             // plural — menu title
        'singular_name' => 'Press & Talks Item',        // singular — used in buttons
        'add_new'       => 'Add New',                   // generic "Add New" button
        'add_new_item'  => 'Add New Press & Talks Item',
        'edit_item'     => 'Edit Press & Talks Item',
        'all_items'     => 'All Press & Talks Items',   // submenu link label
        'menu_name'     => 'Press & Talks',             // top-level sidebar label
    ],
    // ...
] );
```

The keys that matter most for sidebar clarity: `all_items`, `add_new_item`, and `menu_name`. If you only set three, set those three.

---

