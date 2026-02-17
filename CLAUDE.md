# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Obsidian plugin that decorates headings (H1-H6) with numbering/labeling across multiple Obsidian views. Supports ordered (decimal, roman, CJK, etc.), unordered (custom labels per level), independent (per-level counter styles), and splice (combined per-level) decoration modes. Works in reading view, live preview, source mode, outline, Quiet Outline plugin, and file explorer.

## Commands

```bash
# Install dependencies (pnpm only - see pnpm-lock.yaml)
pnpm install

# Development build with watch mode
pnpm dev

# Production build (typecheck + minified bundle)
pnpm build

# Run all tests
pnpm test

# Run a single test file
pnpm mocha test/common/counter.spec.ts

# Lint
pnpm eslint .

# Release (bumps version in package.json, manifest.json, versions.json)
pnpm release
```

## Architecture

### Build System

esbuild bundles `main.ts` -> `main.js` (CJS format). Obsidian APIs, CodeMirror, Electron, and Lezer are externalized (provided by Obsidian at runtime). TypeScript is checked via `tsc -noEmit` during `pnpm build` but not during `pnpm dev`.

### Plugin Entry Point

`main.ts` re-exports `HeadingPlugin` from `components/plugin.ts`. The plugin class:
- Uses a **deep revocable Proxy** on settings to detect changes at any nesting depth and trigger selective re-renders per view type
- Manages separate component lifecycles for each view type (reading, outline, quiet-outline, file-explorer)
- Registers a CodeMirror `ViewPlugin` for editor (live preview + source mode) decorations

### Core Layers

**`common/`** - Pure logic, no Obsidian DOM dependencies (testable):
- `heading.ts` - Markdown heading parser (state machine handling frontmatter, code blocks, math blocks, setext headings)
- `counter.ts` - Counter implementations: `Querier` (tracks heading levels), `OrderedCounter`, `IndependentCounter`, `SpliceCounter`, `UnorderedCounter`
- `data.ts` - Settings types (`HeadingPluginSettings`, `HeadingDecoratorSettings`), defaults, CSS class names, and utility functions
- `options.ts` - Maps `@jsamr/counter-style` presets to i18n display labels

**`components/`** - Obsidian-specific rendering:
- `editor.ts` - CodeMirror 6 extension: `StateField` for decorations + `HeadingEditorViewPlugin` that builds `Decoration.widget` ranges from parsed headings. Handles both live preview and source mode via `editorModeField`
- `reading.ts` / `reading-child.ts` - Reading view: HTML DOM manipulation via `MarkdownPostProcessor`, creates `<span>` decorators on heading elements
- `outline.ts` - Native Obsidian outline panel decorations
- `quiet-outline.ts` - Quiet Outline plugin (third-party) panel decorations using `.n-tree` selectors
- `file-explorer.ts` - File explorer heading decorations using `.nav-file-title` selectors
- `child.ts` - `ViewChildComponent` base for outline/quiet-outline/file-explorer (MutationObserver-based re-rendering)
- `weight.ts` - `HeadingWidget` (CodeMirror `WidgetType`) for editor decorations
- `setting.ts` / `setting-tab.ts` / `suggest.ts` - Settings UI

### Decoration Modes

The `DecoratorMode` type (`types.d.ts`) defines four modes:
- `"orderd"` (note: typo is intentional, kept for backwards compatibility) - Hierarchical numbering (e.g., 1.1, 1.2)
- `"independent"` - Each heading level has its own counter style
- `"splice"` - Like ordered but each level segment uses a different style
- `"unordered"` - Static labels per level (e.g., H1, H2, H3)

### Settings Architecture

Each view type (reading, preview, source, outline, quiet-outline, file-explorer) can use either `commonSettings` or its own `*Settings` override. Per-note control via frontmatter keyword (default: `heading`) or `cssclasses`. Folder and file-regex blacklists for exclusion.

### i18n

Uses `obsidian-plugin-i18n` with locale files in `locales/` (en, zh, zh-TW).

### Tests

Mocha + Chai (BDD style), run via `tsx` loader. Tests cover `common/` modules only (pure logic). Spec files mirror `common/` structure under `test/common/`.
