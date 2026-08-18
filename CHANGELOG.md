# Changelog

All notable changes to the SilverBullet Unlinked Mentions plug.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

- Nothing yet.

## [0.6.1] - 2026-08-18

### Fixed
- **`share.uri` used the `ghr:` scheme pointing at a non-existent release** — the repo has no GitHub releases, so update pulls failed with a 404. Switched to a plain `https://` blob URL (pull mode works against repository files).
- **Config example blocks used `space-lua` fences** — SilverBullet executes any `space-lua` block when a page (including a documentation page) is loaded, so the "disable the widget" example in the docs could silently set `unlinkedMentions.enabled = false`. Example blocks now use a plain `lua` fence and are documentation-only.

## [0.6.0] - 2026-08-12

### Added
- Command `Unlinked Mentions: Link All (This Page)` — converts ALL unlinked mentions of the current page (not limited by `maxResults`), with a confirmation prompt.
- Progress feedback for the space-wide command (notification every 50 pages scanned).
- Configurable snippet context length: `contextLen` (default 100 characters around each mention).
- Refined link-failure notifications: distinguishes "mention exists but only in unsafe locations (code/link/frontmatter)" from "no mention of X in page".

### Fixed
- **Space-wide command used the wrong aliases** — `searchUnlinked` now accepts a `pageText` argument. The full-space command reads each scanned page's own text before extracting its aliases, instead of incorrectly using the currently-open editor page's aliases.
- Removed dead code (`pagesTouched`).

### Changed
- **Breaking:** config namespace renamed `std.widgets.unlinkedMentions` → `unlinkedMentions` (the `std.` prefix belongs to SilverBullet's standard library). Update your `CONFIG` if you set options explicitly.

## [0.5.0] - 2026-08-12

### Added
- Command `Unlinked Mentions: Link All (Full Space)` — converts every unlinked mention across the whole space, with a confirmation prompt.
- Command `Unlinked Mentions: Link All (Full)` — early version of the space-wide command (superseded by 0.6.0's scoped commands).

### Changed
- Rendering now uses pure markdown output to match the native Linked Mentions look.

## [0.4.0] - 2026-08-12

### Changed
- **Snippets are now pure text.** Context excerpts are parsed with SilverBullet's own `markdown.parseMarkdown` and rendered as plain text, so `# headings`, `**bold**`, and `[[links]]` from source pages never leak into the widget as live markdown (previously a heading line in a source page could render as a giant heading inside the widget).
- Removed the native `<details>` fold arrow (it conflicted with the H1 block layout); the heading itself is still clickable to collapse/expand.

## [0.3.0] - 2026-08-12

### Changed
- **English UI** — all tooltips, notifications, and button text are now in English.
- **Visual consistency** — the heading renders as a proper H1 inheriting the theme's heading style, matching Linked Mentions.
- **Breaking:** config namespace renamed `std.widgets.unlinkedMentions` → `unlinkedMentions`.
- **Excluded folders now work both ways**: pages in excluded folders won't appear in mention lists on other pages, AND the widget won't render on pages inside excluded folders.

## [0.2.0] - 2026-08-11

### Changed
- Replaced the checkbox multi-select UI with per-item `Link` buttons plus a bottom `Link All` button (native checkboxes proved unreliable inside CodeMirror widgets).
- Hashtags (`#tag`) are excluded from mention detection and conversion.

## [0.1.0] - 2026-08-11

### Added
- Initial release: displays unlinked mentions (plain-text references to the current page without a `[[wikilink]]`) at the bottom of pages.
- Exact substring verification to eliminate fuzzy-match false positives from Silversearch.
- One-click conversion of mentions to wikilinks.
- Safe replacement: skips frontmatter, code blocks, inline code, existing links, and Markdown link URLs.
- Alias support via frontmatter (`aliases`), converted as `[[page|alias]]` preserving original text.
- Collapsible section (native `<details>`), defaults to expanded.
- Folder exclusion (Library/, System/, templates).
