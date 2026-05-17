# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Type-check and format-check
npm run check

# Auto-format all files
npm run format

# Run tests (tsx native test runner)
npm test

# Run a single test file
npx tsx --test quartz/util/path.test.ts

# Build & serve locally
npx quartz build --serve -d docs

# Full build without serving
npx quartz build -d docs

# Build with verbose logging
npx quartz build --verbose -d docs

# Create a new Quartz site
npx quartz create
```

The CLI entry point is `quartz/bootstrap-cli.mjs`. Commands: `build`, `create`, `update`, `restore`, `sync`.

## Architecture

Quartz is a static site generator for a digital garden — it turns a folder of Markdown files into a static HTML website. The core is a three-stage pipeline:

### Pipeline: filter → parse → emit

1. **Filter** (`quartz/processors/filter.ts`): Runs filter plugins to exclude content that shouldn't be published (e.g. drafts).
2. **Parse** (`quartz/processors/parse.ts`): Two-phase Markdown processing via unified/remark/rehype:
   - Text → Markdown AST (remark-parse + transformer `markdownPlugins`)
   - Markdown AST → HTML AST (remark-rehype + transformer `htmlPlugins`)
   - For large sites, parsing uses worker threads via `workerpool` and `quartz/bootstrap-worker.mjs`.
3. **Emit** (`quartz/processors/emit.ts`): Runs emitter plugins that produce output files (HTML pages, RSS, sitemap, assets, etc.)

### Plugin system (`quartz/plugins/`)

Three plugin types defined in `quartz/plugins/types.ts`:

- **Transformers** — transform content during parsing. Have `textTransform` (raw string), `markdownPlugins` (remark), and `htmlPlugins` (rehype) hooks. Examples: syntax highlighting, wikilinks, frontmatter extraction.
- **Filters** — decide whether a file should be published. Single hook: `shouldPublish(ctx, content) -> boolean`.
- **Emitters** — produce output files. Hook: `emit(ctx, content, resources) -> FilePath[]`. Optionally `partialEmit` for incremental rebuilds. Examples: `ContentPage` (renders each note), `TagPage`, `ContentIndex` (RSS/sitemap).

### Components (`quartz/components/`)

UI components are Preact TSX components with optional `css`, `beforeDOMLoaded`, and `afterDOMLoaded` string resources. The page layout is assembled in `quartz.layout.ts` using a grid: `left` sidebar, `center` (header, content, footer), `right` sidebar. `renderPage.tsx` combines layout components with the HTML AST tree from parsing.

Key component infrastructure:
- `renderPage.tsx`: Renders the Preact page shell, handles transclusions (embedding one note inside another by substituting blockquote nodes in the HTML AST).
- `Body.tsx`: Layout wrapper; uses CSS class-based sidebar toggling.
- `types.ts`: `QuartzComponentProps` — every component receives `{ctx, fileData, cfg, tree, allFiles, externalResources}`.

### Configuration

- `quartz.config.ts` — site-wide config and plugin registration (which transformers, filters, emitters are active).
- `quartz.layout.ts` — UI layout: which components appear in left/right/center areas for single-note pages and list pages.
- `quartz/cfg.ts` — TypeScript types for config and layout.

### Build system

The `build` command uses esbuild to transpile the config + entire Quartz codebase into `quartz/.quartz-cache/transpiled-build.mjs`, then imports and runs it. Watch mode uses chokidar and WebSocket for live reload. Serve mode hosts the output directory with `serve-handler`.

### Key data structures

- `QuartzPluginData` (in `quartz/plugins/vfile.ts`): Metadata attached to each vfile — slug, frontmatter, links, tags, file path, dates.
- `BuildCtx` (in `quartz/util/ctx.ts`): Context object passed through the entire pipeline — carries config, CLI args, file lists.
- `StaticResources` (in `quartz/util/resources.ts`): Collected CSS, JS, and head elements emitted as part of the build.
