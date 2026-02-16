# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A Claude Code plugin collection for e-commerce product scraping. Currently contains one plugin (`taobao-skill`) that scrapes Taobao/Tmall product pages via Chrome DevTools MCP — no Playwright scripts or hardcoded selectors. The plugin uses `evaluate_script` and `take_snapshot` to dynamically extract data by reading `window.__ICE_APP_CONTEXT__` (server-side rendered data).

## Architecture

```
ecommerce-skills/                    # Marketplace root (marketplace.json defines plugin registry)
├── .claude-plugin/marketplace.json  # Plugin registry — lists all plugins with name, source path, description
├── .claude/settings.json            # Enables plugins for this project
├── .mcp.json                        # Project-level MCP server config (user-specific, gitignored)
└── taobao-skill/                    # Individual plugin (Claude Code plugin structure)
    ├── .claude-plugin/plugin.json   # Plugin metadata (name, version, author)
    └── skills/                      # Skills exposed as slash commands and auto-activated by description matching
        └── taobao-product-scraper/
            ├── SKILL.md             # Layered: prerequisites → workflow → extraction knowledge → output format
            └── references/          # Supplementary domain knowledge (volatile/detailed)
                ├── taobao-page-structure.md   # CSS selectors, JS data paths, image URL cleaning
                ├── taobao-login-flow.md       # Multi-factor login detection, QR flow, session persistence
                └── taobao-known-issues.md     # A/B testing, share links, selector drift, rate limiting
```

### Key Concepts

- **Skills** (`skills/*/SKILL.md`): Self-contained units with YAML frontmatter. Each skill is exposed as a slash command and auto-activates when the user's request matches its `description` field. The SKILL.md contains both workflow orchestration and domain knowledge in a layered structure.
- **References** (`skills/*/references/*.md`): Supplementary knowledge files for detailed/volatile information (selectors, known issues) kept separate from the core skill.
- **MCP config** (`.mcp.json`): Not bundled with the plugin — users must create their own at the project level. The skill includes setup instructions if MCP tools are missing.

### Data Extraction Strategy

The plugin follows a JS-first → DOM-fallback approach:
1. **Primary**: `evaluate_script` reads `window.__ICE_APP_CONTEXT__.loaderData.home.data.res` for all product data in a single call
2. **Fallback**: `take_snapshot` reads the accessibility tree when JS context is unavailable
3. **A/B handling**: Taobao randomly serves simplified pages — detect via `_version` field and retry (max 2 reloads)

## Adding a New E-Commerce Plugin

1. Create a new directory at the repo root (e.g., `jd-skill/`)
2. Add `.claude-plugin/plugin.json` with name, description, version
3. Add `skills/` directory with a SKILL.md (layered: prerequisites → workflow → knowledge → output)
4. Register the plugin in `.claude-plugin/marketplace.json` under `plugins[]`
5. Enable it in `.claude/settings.json` under `enabledPlugins`

## Conventions

- All skill files are Markdown with YAML frontmatter
- Reference files contain volatile data (selectors, known issues) that changes frequently — keep them separate from core skill logic
- Image URLs from Taobao CDN must be cleaned (remove `_qNN`, `_NNxNN`, `.webp` suffixes) — patterns documented in `taobao-page-structure.md`
- Always use desktop URLs (`detail.tmall.com`), never mobile (`detail.m.tmall.com`)
- Share links must be resolved to clean URLs: `https://detail.tmall.com/item.htm?id={product_id}`
