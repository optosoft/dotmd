# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

**dotmd** is a single-page, installable PWA Markdown editor. The entire application
(styles, markdown extensions, and React UI) lives in one file: `index.html`. There is
no build step, package manager, bundler, or test suite — React, Tailwind, marked.js,
KaTeX, Mermaid, PlantUML encoder, and highlight.js are all loaded from CDNs, and the
app's React components are written in-browser using Babel's standalone transform
(`<script type="text/babel">`).

## Development workflow

- There is no build/install/lint/test command — edit `index.html` directly.
- To preview changes, open `index.html` in a browser, or serve the directory with any
  static file server (a server is needed for the PWA `manifest.json` to load correctly
  over `http://`/`https://`, e.g. `npx serve .` or `python -m http.server`).
- Since everything runs via in-browser Babel, syntax errors in the `<script type="text/babel">`
  block fail silently/at runtime in the browser console — check there after edits.

## Architecture

Everything is implemented inside the single `<script type="text/babel">` block at the
bottom of `index.html`, structured as follows:

- **Markdown pipeline**: `marked` is configured once at module scope with a custom
  `emojiExtension` (`:shortcode:` → emoji via `EMOJI_MAP`) and a custom heading renderer
  that slugifies headings into `id` attributes (used by the Table of Contents).
  `processExtendedMarkdown(md)` is a pre-processing pass run before `marked.parse()` that:
  - Extracts YAML front-matter into a rendered metadata table.
  - Converts ` ```plantuml ` blocks into PlantUML server image URLs (via `plantuml-encoder`).
  - Temporarily shields fenced/inline code blocks (so they aren't mangled) while it
    rewrites footnotes (`[^id]`) and simple definition lists, then restores the code blocks.
- **Single root component**: `App()` holds all state — there are no other components or
  files. Key state groups:
  - **Files/workspace**: `files` (array of `{id, name, content}`) and `activeFileId`,
    persisted to `localStorage` under `dotmd_files_v1` / `dotmd_active_file_id`
    (migrates from a legacy single-document key `dotmd_v2` on first load).
  - **Per-file undo/redo**: a `Map` (`historyMap`, keyed by file id) of `{past, future,
    lastSaved}`, debounced via `saveHistoryTimer` (500ms) so rapid typing collapses into
    one history entry.
  - **UI/layout**: `layout` (`'editor' | 'split' | 'preview'`), `showSidebar`, `showToc`,
    `showSettings`, `showHelp`, `isFocusMode`, `isMobile`/`isResizing` for responsive and
    resizable-sidebar behavior, plus `darkMode` and `customCSS` (both persisted to
    `localStorage` and injected via DOM mutation in `useEffect`s).
  - **Emoji autocomplete**: `emojiPicker` state, triggered while typing `:` in the editor,
    positioned using the `getCaretCoordinates()` helper.
- **Editor pane**: a transparent `<textarea>` overlaid on a `<pre><code>` element that
  shows the highlight.js-highlighted markdown — the two are kept in sync via `onScroll`
  (`handleScroll`) so the highlighted backdrop scrolls with the textarea.
- **Preview pane**: renders `marked.parse(processExtendedMarkdown(markdown))` via
  `dangerouslySetInnerHTML`. After each render, `renderFeatures()` runs to:
  - Typeset math with KaTeX (`renderMathInElement`, `$...$` / `$$...$$`).
  - Render Mermaid diagrams (`.language-mermaid` blocks) via `mermaid.run()`.
  - Apply highlight.js to remaining code blocks and inject a "copy to clipboard" button.
  - In split layout (desktop only), preview scroll position is synced proportionally to
    editor scroll via `handleScroll`.
- **Export**: `doExport(type)` supports `md`, `txt`, `html` (self-contained document with
  `marked.parse(processExtendedMarkdown(markdown))`), and `pdf` (switches to preview-only
  layout and calls `window.print()` — print-specific CSS is in the `@media print` block).
- **Help drawer**: the in-app documentation is itself a markdown string (`helpMarkdown`)
  rendered through the same `processExtendedMarkdown` + `marked.parse` + `renderFeatures`
  pipeline as the main preview.

## Conventions to follow when editing

- Keep new functionality within `index.html` consistent with the existing pattern of
  CDN-loaded dependencies and Babel-transformed JSX — do not introduce a build step
  unless explicitly asked.
- Anything user-configurable that should survive a reload must be persisted to
  `localStorage` (follow the `dotmd_*` key naming convention) and loaded back in an
  `useEffect`/initializer.
- Preview-affecting styling should target `.preview-area` classes in the `<style>` block
  (and is the documented extension point for the Custom CSS settings feature).
- When adding new markdown syntax extensions, integrate them into
  `processExtendedMarkdown` (for pre-processing) or as a `marked` extension (for inline
  tokenization), and document them in `helpMarkdown` so they appear in the in-app Help drawer.
