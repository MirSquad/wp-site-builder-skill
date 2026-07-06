# WordPress Site Builder — Claude Code Skill

A Claude Code skill for building production-quality WordPress sites from scratch. Covers the full process from static HTML/CSS prototype through to a production-ready WordPress theme, with explicit standards for content management architecture, performance, accessibility, and agent-readiness.

## What this is

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) — a `SKILL.md` file plus a set of reference files that teach Claude how to build WordPress sites correctly. When installed, Claude follows a two-phase process: build a clean static prototype first, then convert it to a fully functional WordPress theme, consulting the relevant reference file at each step.

## Structure

```
SKILL.md                              Overview, non-negotiables, and pointers to each reference
references/
  phase1-prototype.md                 Static HTML/CSS/JS prototype standards
  phase2-wordpress.md                 WordPress theme conversion: file structure, functions.php,
                                       templates, forms, menus, sanitization, versioning
  content-management.md               Content management architecture — read before writing templates
  performance-accessibility.md        Performance, accessibility, and agent-readiness standards
```

`SKILL.md` stays short and links out to the reference files, which Claude reads at the point it needs them (e.g. `content-management.md` before writing a single template file).

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
- Native `wp_mail()` contact forms, mobile navigation, lightboxes, menus
- Sanitization cheatsheet, version management, dead code patterns to avoid

### Content Management Architecture
- **The golden rule**: never hardcode content in template files
- Content management matrix — where every type of content should live
- Native WordPress first (page content > featured images > ACF > Customizer > CPTs)
- ACF fields registered in code, not the UI
- Custom post types with proper labels
- Gutenberg blocks vs. Classic Editor block, XML import files for initial content

### Performance
- Image optimization (sizing, compression, `fetchpriority`, `loading="lazy"`)
- Hero image preloading via `wp_head` (not body)
- Google Fonts async loading (non-render-blocking `media="print"` pattern)
- Dark mode flash prevention
- CSS/JS delivery (defer, dequeue unused block library)
- CDN caching gotchas on managed WordPress hosting
- Speculation Rules + instant.page for perceived speed
- PageSpeed score targets and CSS line clamping gotchas in flex containers

### Accessibility (WCAG 2.1 AA)
- Color contrast requirements with separate accent-text variables
- Keyboard navigation and focus indicators
- Skip links, landmarks, ARIA labels
- Modal/dialog patterns (use `aria-label`, not `aria-labelledby` on hidden elements)
- bfcache compatibility

### Agent Readiness
- Fundamentals to add by default on every build, not just on request: feed discovery, theme-color/color-scheme meta, `prefers-reduced-motion`
- When to reach for standard plugins vs. hand-rolling
- Verification steps before calling a build done

## Installation

### Option 1: Copy the skill files

```bash
mkdir -p .claude/skills/wp-site-builder
cp SKILL.md .claude/skills/wp-site-builder/
cp -r references .claude/skills/wp-site-builder/
```

### Option 2: Reference from your CLAUDE.md

```markdown
## Skills
- See `.claude/skills/wp-site-builder/SKILL.md` for WordPress site building standards
```

## Usage

Once installed, Claude will follow this skill when you ask it to:
- Build a WordPress site or theme
- Create a personal or portfolio site
- Convert a design to WordPress
- Build any content-managed website

The skill enforces the two-phase approach: prototype first, then WordPress conversion. This produces better themes because design decisions are resolved before CMS complexity is introduced.

## Keeping the skill current

The skill includes a self-improvement loop: at the end of every working session, Claude is instructed to ask whether anything discovered during the build — hosting gotchas, patterns that worked or didn't, performance/accessibility techniques, security patterns — should be appended to the relevant reference file. Client-specific content and one-off decisions are explicitly excluded. See the "Updating This Skill" section in `SKILL.md` for the full rule.

## Origin

Built and maintained by [Miriam Schwab](https://miriamschwab.me), refined through building multiple production WordPress sites. Every pattern — from the content management golden rule to the CSS line clamping gotcha — was discovered through real projects and real debugging sessions.

## License

MIT
