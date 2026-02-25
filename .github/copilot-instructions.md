# Copilot Instructions for personal-blog

## Project Overview

This is a personal blog built with **Hugo** (v0.154.5+), deployed to **Vercel**. The blog uses the **hugo-bearblog** theme and includes Giscus-powered comments.

**Key Links:**
- Website: https://krusty.blog/
- Theme: hugo-bearblog (git submodule in `src/themes/`)
- Comments: Giscus (repo: Krusty93/personal-blog)

## Architecture & Key Concepts

### Hugo Project Structure
- **Root**: `/src/` contains the entire Hugo site (config, content, layouts, themes)
- **Content**: `/src/content/posts/` - blog posts in Markdown with TOML front matter
- **Layouts**: `/src/layouts/` - custom HTML overrides (partials for header, footer, comments)
- **Theme**: `/src/themes/hugo-bearblog/` - managed as git submodule
- **Output**: `/public/` - generated static HTML (do not edit directly)
- **Static Assets**: `/src/static/images/` - referenced as `/images/` in HTML

### Configuration
- **hugo.toml**: Primary config file in `/src/` (NOT repo root)
  - Sets baseURL, theme, language, metadata
  - Disables taxonomy kinds (tags/categories not used as separate pages)
  - Configures Giscus comment integration with specific repo IDs
  - Output formats: RSS feeds for home + sections
- **vercel.json**: Build environment specifying Hugo version

## Developer Workflows

### Creating Posts
```bash
# Create new post (run from /src)
cd src
hugo new posts/001-post-name.md

# Alternative from repo root
hugo new --source src posts/001-post-name.md

# IMPORTANT: Posts are created in /src/content/posts/
# The archetype in /src/archetypes/default.md sets:
# - Front matter: date, draft=true, auto-formatted title
# - Default draft status requires removing/setting draft=false to publish
```

### Building & Testing
```bash
# Build static site
hugo --source src --destination public

# Development server (with drafts visible)
hugo server --source src --draft
# Access at http://localhost:1313/

# Devcontainer debug server (bind all interfaces)
hugo server --source src --bind 0.0.0.0 --baseURL /
```

### Content Workflow
- Posts should be in `/src/content/posts/` as Markdown files
- Front matter uses TOML format (+++...+++)
- Required fields: `title`, `date` (auto-populated), `draft` (must be false to publish)
- Permalinks configured as `/:slug/` for cleaner URLs
- Posts must have weight/ordering via menu system if needed

### Deployment
- Vercel automatically builds on push to main
- Build command runs: `hugo --source src --destination public`
- Ensure Hugo v0.154.5 compatibility in any version upgrades

## Project-Specific Conventions

### Branch Naming
- Avoid `feature/` or `fix/` prefixes for branch names
- Use direct descriptive names (e.g., `update-theme`, `add-blog-post`, `fix-styling`)

### Post Front Matter Format
All posts use TOML with this structure:
```toml
+++
date = '2025-01-28T00:00:00Z'
draft = false
title = 'Post Title'
+++
```

### Custom Layouts & Partials
Located in `/src/layouts/partials/`, these files customize the theme:

- **header.html** – Site header with Vercel analytics scripts and navigation link
  - Integrates Vercel Speed Insights and Web Analytics
  - Renders site title and includes nav partial
  
- **footer.html** – Footer with social links and attribution
  - Renders links from `params.socialLinks` in hugo.toml (GitHub, LinkedIn, RSS, InfoQ)
  - Custom SVG icons for each platform
  - Attribution to Hugo Bear theme and Icons8
  
- **comments.html** – Giscus comment system integration
  - Uses `params.giscus` config from hugo.toml for all repository/category IDs
  - Loads Giscus script with theme/language settings
  - Auto-included in single post template

**Modification Pattern**: Edit partials in `/src/layouts/partials/` to override theme defaults. Do not edit theme files directly.

### Styling & Theme Customization
- Theme CSS/JS is in `src/themes/hugo-bearblog/` (don't edit directly; use Hugo override system)
- Override theme files by placing same-path files in `/src/layouts/` or `/src/static/`
- Code highlighting uses "friendly" style with line numbers

### Taxonomy & Organization
- **Disabled**: Hugo taxonomy (tags/categories as separate pages)
- **Alternative**: Use post-level tagging if needed, but no auto-generated taxonomy pages
- Posts are organized purely by publication date

## Integration Points

### External Services
- **Vercel**: Deployment platform (via vercel.json)
- **GitHub**: Repository hosting + Giscus comment backend
- **Giscus**: Comment system (configured with repo/category IDs in hugo.toml)

### RSS & Feeds
- RSS feeds generated for:
  - Home page: `/feed/`
  - Post section: `/posts/feed/`
- Feed URL accessible at `/feed.xml` and `/posts/feed.xml`

## Critical Notes

1. **Hugo Config Location**: Always reference `/src/hugo.toml`, not `/hugo.toml`
2. **Theme is Git Submodule**: Theme is managed via git submodule at `src/themes/hugo-bearblog/`
  - Preferred update command: `git submodule update --remote src/themes/hugo-bearblog`
   - Don't commit theme changes; customize only via partials in `/src/layouts/`
  - Ignore legacy `.gitmodules` entry for `src/themes/bearblog` (canonical path is `src/themes/hugo-bearblog`)
3. **Draft Status**: Posts with `draft = true` won't be published; must explicitly set to `false`
4. **Public Directory**: Generated output—never manually edit; always regenerate via Hugo
5. **Vercel Build**: Ensure `src/` directory is preserved for builds to work correctly

## Theme Maintenance

### Updating hugo-bearblog Submodule
```bash
# Check for updates
cd src/themes/hugo-bearblog
git fetch origin

# Update to latest version on default branch
git pull origin master

# Return to root and commit
cd /workspaces/personal-blog
git add src/themes/hugo-bearblog
git commit -m "Update hugo-bearblog theme"
```

### Testing Theme Updates
After updating the submodule:
```bash
hugo server --source src --draft
```
Verify all custom partials still work (header, footer, comments) and regenerate site if compatible.
