# Content Management Architecture

**Read this before writing any WordPress template.**

The single most important architectural decision in a WordPress theme is: where does the content live? Get this wrong and the site looks right on launch but becomes unmaintainable immediately after — every content update requires a developer.

The rule is simple: **content belongs to the person who owns it, managed in the place they'd naturally expect to find it.**

---

## The Golden Rule

**Never hardcode content in template files.**

This includes: page titles, introductory paragraphs, descriptions, headings, taglines, bio text, photo URLs, social media links, contact information — anything a site owner might want to update.

The only things that should be hardcoded in templates are:
- Structural labels that are part of the UI, not the content (e.g. `<span class="section-label">about</span>`)
- Static UI text that would never change regardless of who runs the site (e.g. "back to writing")
- Fallback values for fields that haven't been filled in yet

---

## The Content Management Matrix

Use this to decide where each piece of content should live:

| Content type | Where it lives | How to edit |
|---|---|---|
| Page title | WordPress page title field | Page editor — Title input |
| Page body text | WordPress page content editor | Page editor — Block editor |
| Page featured photo | WordPress featured image | Page editor — Featured Image panel |
| Structured repeating data (timeline, now items, etc.) | ACF fields on the page | Page editor — ACF metabox below content |
| Short text on a specific page (tagline, eyebrow) | ACF field on that page | Page editor — ACF metabox in sidebar |
| Global settings (social links, global phone number) | Customizer | Appearance → Customize |
| Posts, media appearances, events | Custom post types | Their own sidebar menu items |
| Archive/listing page descriptions | The page's own content editor | Page editor — Block editor |

---

## Native WordPress First

Always use native WordPress mechanisms before reaching for options, custom tables, or theme mods for simple cases. In order of preference:

1. **Page content** (`the_content()`) — for any text that belongs to a specific page
2. **Page title** (`the_title()`) — for the page heading
3. **Featured image** (`get_the_post_thumbnail_url()`) — for page-specific photography
4. **ACF fields on the page** — for structured data that doesn't fit the content editor
5. **Customizer** (`get_theme_mod()`) — for site-wide settings
6. **Custom post types** — for repeating structured content (appearances, timeline items, events)

Only use `get_option()` as a last resort for things that genuinely don't belong anywhere else.

---

## Page Content

For any page with body text, use `the_content()`. Do not put that text in Site Options or a custom field. The person editing the site expects to find page content in the page editor.

```php
// CORRECT
the_post();
the_content();

// WRONG
echo get_option('ms_about_intro');
```

For pages that use custom templates (Template Name declarations), call `the_post()` at the top of the template to set up the post object, then use `the_content()`, `the_title()`, `get_the_post_thumbnail_url()`, etc. normally.

```php
<?php
/**
 * Template Name: About
 */
get_header();
the_post();
?>
<h1><?php the_title(); ?></h1>
<div><?php the_content(); ?></div>
```

---

## Featured Images

For any page-specific photo (a headshot on the About page, a hero image on the homepage), use the native WordPress featured image system. This means:

1. Ensure `add_theme_support( 'post-thumbnails' )` is in `functions.php`
2. In the template, check for and display the featured image:

```php
if ( has_post_thumbnail() ) {
    $photo_url = get_the_post_thumbnail_url( get_the_ID(), 'large' );
} else {
    $photo_url = get_template_directory_uri() . '/assets/images/fallback.jpg';
}
```

Never put a photo URL in a custom field when a featured image will do. Featured images are a native WordPress concept that editors already understand.

---

## ACF Fields — When and How

Use ACF fields when the content is structured in a way the block editor can't handle cleanly — short text snippets that appear in specific positions, or repeating structured items.

**Register fields in code, not via the ACF UI.** Fields registered in the UI are stored in the database and tied to that specific install. Fields registered in code are part of the theme and work immediately on any install.

Always use `add_action('init', ..., 20)` — not `add_action('acf/init', ...)`. The `init` hook at priority 20 ensures `get_page_by_path()` and `get_option()` are available when the group registers. `acf/init` fires before these functions are reliable and is only safe for CPT-based rules with no page-ID lookups. Using `init, 20` for everything avoids the inconsistency.

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
                'name'  => 'field_name',  // this is what get_field() uses
                'type'  => 'text',
            ],
        ],
        'location' => [ [ [
            'param'    => 'post',
            'operator' => '==',
            'value'    => $page_id,  // specific page ID
        ] ] ],
        'position' => 'side',  // 'normal' for below content, 'side' for sidebar
    ] );
}, 20 );
```

**Location rules:**
- For a specific page: use `'param' => 'post', 'value' => $page_id`
- For all pages using a template: use `'param' => 'page_template', 'value' => 'page-about.php'` — BUT this only works if the template is explicitly selected in the Template dropdown, not if it's applied automatically by filename. If you're applying templates by filename (e.g. `page-about.php` auto-applies to the About page), use page ID instead.
- For a custom post type: use `'param' => 'post_type', 'value' => 'ms_media'`

---

## Custom Post Types

Use custom post types for any repeating structured content that has its own data model: media appearances, timeline items, events, team members, testimonials, etc.

Register in `functions.php`:

```php
function ms_register_cpts() {
    register_post_type( 'ms_media', [
        'labels'          => [
            'name'      => 'Press & Talks',
            'menu_name' => 'Press & Talks',
        ],
        'public'          => true,
        'show_ui'         => true,
        'show_in_menu'    => true,
        'show_in_rest'    => true,
        'menu_icon'       => 'dashicons-microphone',
        'supports'        => [ 'title', 'editor', 'thumbnail' ],
        'rewrite'         => [ 'slug' => 'media' ],
    ] );
}
add_action( 'init', 'ms_register_cpts' );
```

**CPT naming:** Avoid names that clash with WordPress core. `media` is taken. `appearance` is dangerously close to `Appearance` in the menu. Prefix CPT slugs with something specific to the project (e.g. `ms_media`, `ms_timeline`).

**For CPTs that should not have public-facing URLs** (like events that only appear in a homepage section): set `'public' => false, 'show_ui' => true`. This hides them from sitemaps, search, and direct URL access while still making them manageable in the admin.

---

## The Customizer for Global Settings

Site-wide settings — social media URLs, a global phone number, a press kit URL — belong in the Customizer, not in a custom options page.

```php
function ms_customize_register( $wp_customize ) {
    $wp_customize->add_section( 'ms_social_links', [
        'title'    => 'Social Links',
        'priority' => 30,
    ] );
    $wp_customize->add_setting( 'ms_twitter_url', [
        'default'           => '',
        'sanitize_callback' => 'esc_url_raw',
    ] );
    $wp_customize->add_control( 'ms_twitter_url', [
        'label'   => 'Twitter / X URL',
        'section' => 'ms_social_links',
        'type'    => 'url',
    ] );
}
add_action( 'customize_register', 'ms_customize_register' );
```

Retrieve with `get_theme_mod( 'ms_twitter_url', '' )`.

---

## Archive and Listing Pages

For pages that show a list of posts (a blog, a writing page), the description that appears above the listing should come from the page's own content editor, not from an option. WordPress designates a page as the "posts page" via Settings → Reading. You can get that page's content:

```php
$posts_page_id = get_option( 'page_for_posts' );
$posts_page    = get_post( $posts_page_id );
$description   = apply_filters( 'the_content', $posts_page->post_content );
```

---

## What NOT to Put in a Custom Options Page

A custom options page (added via `add_menu_page()`) is appropriate only for genuinely global configuration that has no natural home. It is NOT appropriate for:

- Page-specific text (use the page editor)
- Page photos (use featured images)
- Social links (use Customizer)
- Per-post structured data (use ACF fields or CPT fields)

If you find yourself building a large options page with fields for every piece of content on every page, that's a signal the content management architecture is wrong. Move the content to where it belongs.

A good custom options page at the end of a project might have one or two fields that genuinely have nowhere else to go. If it has ten fields, reconsider.

---

## Writing ACF Field Values Programmatically

When setting ACF field values via REST API, MCP tools, WP-CLI, or any code path that writes directly to post meta (rather than through the ACF `update_field()` function), ACF requires **two meta entries per field**:

1. The value itself: `meta_key => value`
2. A companion reference entry: `_meta_key => field_key` (the ACF field key, e.g. `field_abc123`)

Without the companion `_fieldname` entry, ACF doesn't recognize the value as belonging to its field group. The value will be in the database but the field will appear empty in the wp-admin editor.

**Example — correct:**
```php
update_post_meta( $post_id, 'media_platform', 'Podcast Name' );
update_post_meta( $post_id, '_media_platform', 'field_ms_media_platform' );
```

**Example — incorrect (value saved but field appears empty):**
```php
update_post_meta( $post_id, 'media_platform', 'Podcast Name' );
// missing: update_post_meta( $post_id, '_media_platform', 'field_ms_media_platform' );
```

**How to find the field key:** Check the meta of an existing post that has the field populated (via `get_post_meta()` or a `get-post-meta` MCP ability). Every `_fieldname` entry holds the ACF field key for that field.

When using ACF's own `update_field()` function, it writes both entries automatically — this only applies when bypassing ACF's API.

---

## Gutenberg Blocks vs Classic Editor Block

When creating post content programmatically, the way content is passed determines whether it lands as native Gutenberg blocks or as a single Classic Editor block.

**Native Gutenberg blocks** — what you want:
- Use a tool or API parameter that explicitly converts markdown or structured content to block markup before saving
- If the MCP ability or REST endpoint has a `markdown` parameter, use it — this triggers server-side conversion to native blocks
- Or pass raw Gutenberg block HTML directly (e.g. `<!-- wp:paragraph --><p>Text</p><!-- /wp:paragraph -->`)

**Classic Editor block** — what to avoid:
- Passing plain text or markdown as the raw `content` field wraps everything in a single `<!-- wp:freeform -->` Classic Editor block
- This looks identical on the front end but is much harder to edit in the block editor and doesn't support block-level styles, patterns, or reuse

**Practical rule:** If a `create-post` ability or REST call accepts a `markdown` parameter, always prefer it over `content`.

---

## XML Import Files for Initial Content

For structured content that needs to exist on launch (timeline items, sample media appearances, initial blog posts), generate WordPress WXR-format XML import files. These allow the site owner to populate a fresh install instantly via Tools → Import → WordPress.

**Critical:** XML import files are separate from the theme zip. The theme zip goes to Appearance → Themes → Upload. The XML files go to Tools → Import. They should be delivered as separate files with clear instructions.

Import file structure:
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<rss version="2.0" xmlns:content="..." xmlns:wp="...">
<channel>
  <wp:wxr_version>1.2</wp:wxr_version>
  <item>
    <title>Item Title</title>
    <content:encoded><![CDATA[Item content here.]]></content:encoded>
    <wp:post_type>ms_timeline</wp:post_type>
    <wp:status>publish</wp:status>
    <wp:menu_order>1</wp:menu_order>
    <wp:postmeta>
      <wp:meta_key>timeline_year</wp:meta_key>
      <wp:meta_value>2024</wp:meta_value>
    </wp:postmeta>
  </item>
</channel>
</rss>
```

---

