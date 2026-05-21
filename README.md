# WordPress Site Builder — Claude Code Skill

A Claude Code skill for building production-quality WordPress sites from scratch. Covers the full process from static HTML/CSS prototype through to a production-ready WordPress theme, with explicit standards for content management architecture, performance, and accessibility.

## What this is

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) — a markdown file that teaches Claude how to build WordPress sites correctly. When installed, Claude follows a two-phase process: build a clean static prototype first, then convert it to a fully functional WordPress theme.

## What it covers

### Phase 1: Static Prototype
- HTML/CSS/vanilla JS standards (no frameworks)
- CSS custom properties, dark mode, responsive design
- Semantic HTML, skip links, heading hierarchy
- Image optimization and LCP handling
- Completeness checklist before WordPress conversion

### Phase 2: WordPress Theme
- Complete theme file structure
- `functions.php` organization (setup, enqueue, CPTs, ACF, helpers)
- `header.php` and `footer.php` with `wp_head()`, `wp_body_open()`, `wp_footer()`
- Template page patterns with `the_post()` setup

### Content Management Architecture
- **The golden rule**: never hardcode content in template files
- Content management matrix — where every type of content should live
- Native WordPress first (page content > featured images > ACF > Customizer > CPTs)
- ACF fields registered in code, not the UI
- Custom post types with proper labels
- Sanitization cheatsheet (`wp_unslash()` before sanitizing)

### Performance
- Image optimization (sizing, compression, `fetchpriority`, `loading="lazy"`)
- Hero image preloading via `wp_head` (not body)
- Google Fonts async loading (non-render-blocking `media="print"` pattern)
- Dark mode flash prevention
- CSS/JS delivery (defer, dequeue unused block library)
- PageSpeed score targets
- CSS line clamping gotchas in flex containers

### Accessibility (WCAG 2.1 AA)
- Color contrast requirements with separate accent-text variables
- Keyboard navigation and focus indicators
- Skip links, landmarks, ARIA labels
- Modal/dialog patterns (use `aria-label`, not `aria-labelledby` on hidden elements)
- bfcache compatibility

## Installation

### Option 1: Copy the skill file

```bash
mkdir -p .claude/skills
cp SKILL.md .claude/skills/wp-site-builder.md
```

### Option 2: Reference from your CLAUDE.md

```markdown
## Skills
- See `.claude/skills/wp-site-builder.md` for WordPress site building standards
```

## Usage

Once installed, Claude will follow this skill when you ask it to:
- Build a WordPress site or theme
- Create a personal or portfolio site
- Convert a design to WordPress
- Build any content-managed website

The skill enforces the two-phase approach: prototype first, then WordPress conversion. This produces better themes because design decisions are resolved before CMS complexity is introduced.

## Origin

Built and maintained by [Miriam Schwab](https://miriamschwab.me), refined through building multiple production WordPress sites. Every pattern — from the content management golden rule to the CSS line clamping gotcha — was discovered through real projects and real debugging sessions.

## License

MIT
