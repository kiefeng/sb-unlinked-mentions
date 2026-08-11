# SilverBullet Unlinked Mentions

A [SilverBullet](https://silverbullet.md) plug that displays **unlinked mentions** (also known as "potential links" or "linked references without links") at the bottom of your pages, similar to Obsidian's Unlinked Mentions panel.

## What it does

When viewing a page, the widget searches your entire space for pages that mention the current page's name (or its aliases) in plain text but **don't** have a `[[wikilink]]` to it. It then shows them in a collapsible section below the built-in Linked Mentions.

Each result includes:
- A clickable link to the mentioning page
- The alias that was matched (if different from the page name)
- A context excerpt showing the surrounding text
- A **`Link` button** to convert that mention into a proper `[[wikilink]]`
- A **"Link All"** button to batch-convert all visible results

## Features

- 🔍 **Full-text search** powered by [Silversearch](https://github.com/MrMugame/silversearch)
- 🔗 **One-click linking** — convert plain text mentions to `[[wikilinks]]`
- 🛡️ **Safe replacement** — skips frontmatter, code blocks, inline code, existing links, and URLs
- 🏷️ **Alias support** — reads `aliases` from frontmatter, preserves original text with `[[page|alias]]` syntax
- 🚫 **Tag exclusion** — `#hashtag` mentions are never treated as unlinked mentions or converted
- 📂 **Folder exclusion** — ignores Library/, System/, template/ by default
- 📁 **Collapsible** — native `<details>` element, defaults to expanded
- 🌏 **Chinese search** — works best with the [silversearch-chinese-tokenizer](https://github.com/LelouchHe/silversearch-chinese-tokenizer)
- 🪶 **Zero build** — pure Space Lua, no compilation needed

## Requirements

- [SilverBullet](https://silverbullet.md) v2.x
- [Silversearch](https://github.com/MrMugame/silversearch) (full-text search library) — install via the Configuration Manager first

For Chinese-language spaces, also install:
- [silversearch-chinese-tokenizer](https://github.com/LelouchHe/silversearch-chinese-tokenizer) (jieba-based Chinese tokenizer)

## Installation

### Option A: Manual (simplest)

1. Download [`Unlinked Mentions.md`](https://github.com/kiefeng/sb-unlinked-mentions/blob/main/Library/kiefeng/Unlinked%20Mentions.md) from this repository
2. Place it anywhere in your SilverBullet space as a `.md` file (e.g. `Library/Unlinked Mentions.md` or any folder you prefer)
3. If you place it somewhere other than `Library/kiefeng/Unlinked Mentions.md`, update the `name` field in the file's frontmatter to match its actual path
4. Reload SilverBullet (Ctrl+Shift+R)

The widget automatically appears at the bottom of pages that have unlinked mentions.

### Option B: Library Manager

If you use the [Configuration Manager](https://silverbullet.md/Library%20Manager), add this repository as a library source and install from there. The file will be placed automatically under `Library/kiefeng/`.

## Configuration

You can customize the widget in your `CONFIG` page:

```space-lua
config.set("std.widgets.unlinkedMentions", {
  enabled = true,
  maxResults = 30,         -- Maximum results to display
  minTermLength = 2,       -- Minimum search term length (single characters are ignored)
  defaultOpen = true,      -- Expand the section by default (set false to collapse)
  excludeFolders = {       -- Folders to exclude from search
    "Library/",
    "System/",
    "template/",
    "Template/"
  }
})
```

To disable:
```space-lua
config.set("std.widgets.unlinkedMentions.enabled", false)
```

## How it works

1. Collects search terms: current page name + any `aliases` from frontmatter
2. Searches using Silversearch for each term
3. Filters out: the current page, already-linked pages, excluded folders
4. Performs **exact substring verification** on each result's excerpt or full text to eliminate fuzzy-match false positives
5. Displays confirmed matches sorted by relevance score

The substring verification is important because Silversearch uses fuzzy matching by default, which can produce false positives (e.g., searching "Journal/2026-08-11" might match "Journal/2026-08-07" due to shared tokens). The widget double-checks every result to ensure the exact term appears in the text.

## Linking safety

The `Link` button only replaces the **first safe occurrence** in each page. It automatically skips:

- YAML frontmatter (between `---` markers)
- Fenced code blocks (```` ``` ````)
- Inline code (`` `code` ``)
- Text already inside `[[wikilinks]]`
- Markdown link URLs `[text](url)`

This prevents corrupting code examples, breaking existing links, or over-linking.

## License

MIT
