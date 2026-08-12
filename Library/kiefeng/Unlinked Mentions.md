---
name: Library/kiefeng/Unlinked Mentions
tags: meta/library
share.uri: "ghr:kiefeng/sb-unlinked-mentions@main/Library/kiefeng/Unlinked Mentions.md"
share.mode: pull
---

# Unlinked Mentions Widget

```space-style
.sb-unlinked summary {
  cursor: pointer;
  user-select: none;
}
.sb-unlinked-count {
  font-weight: normal;
  opacity: 0.6;
}
.sb-unlinked-list {
  padding-top: 0.25rem;
}
.sb-unlinked-item {
  padding: 1px 0;
}
.sb-unlinked-item-head {
  display: flex;
  align-items: center;
  gap: 6px;
}
.sb-unlinked-link {
  cursor: pointer;
  color: var(--ui-link-color);
  text-decoration: none;
}
.sb-unlinked-link:hover {
  text-decoration: underline;
}
.sb-unlinked-term {
  opacity: 0.55;
  font-size: 0.9em;
}
.sb-unlinked-link-one {
  font-size: 0.8em;
  padding: 0 4px;
  cursor: pointer;
  background: none;
  border: none;
  color: var(--ui-accent-color);
  opacity: 0.4;
}
.sb-unlinked-link-one:hover {
  opacity: 1;
  text-decoration: underline;
}
.sb-unlinked-excerpt {
  font-size: 0.85em;
  opacity: 0.5;
  padding-left: 8px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.sb-unlinked-actions {
  padding-top: 4px;
}
.sb-unlinked-btn {
  font-size: 0.85em;
  padding: 2px 8px;
  cursor: pointer;
  background: none;
  border: 1px solid var(--ui-accent-color);
  border-radius: 3px;
  color: var(--ui-accent-color);
  opacity: 0.6;
}
.sb-unlinked-btn:hover {
  opacity: 1;
}
.sb-unlinked-more {
  font-size: 0.85em;
  opacity: 0.5;
}
```

```space-lua
-- priority: 10
widgets = widgets or {}

-- ============ Configuration ============
config.define("unlinkedMentions", {
  type = "object",
  properties = {
    enabled = schema.boolean(),
    maxResults = schema.number(),
    minTermLength = schema.number(),
    defaultOpen = schema.boolean(),
    excludeFolders = schema.array("string"),
  }
})

config.set("unlinkedMentions", {
  enabled = true,
  maxResults = 30,
  minTermLength = 2,
  defaultOpen = true,
  excludeFolders = {"Library/", "System/", "template/", "Template/"}
})

-- ============ Helpers ============

local function startsWith(str, prefix)
  return string.sub(str, 1, #prefix) == prefix
end

local function shouldExclude(pageName, excludeFolders)
  if not pageName then return true end
  for _, folder in ipairs(excludeFolders) do
    if startsWith(pageName, folder) then return true end
  end
  return false
end

local function getExcerptText(r)
  if r.excerpts and type(r.excerpts) == "table" and #r.excerpts > 0 then
    return r.excerpts[1].excerpt or ""
  end
  return ""
end

local function getAliases(pageText)
  local terms = {}
  local ok, fm = pcall(index.extractFrontmatter, pageText)
  if ok and fm and fm.frontmatter then
    local aliases = fm.frontmatter.aliases or fm.frontmatter.alias
    if aliases then
      if type(aliases) == "string" then
        for alias in string.gmatch(aliases, "[^,]+") do
          local trimmed = string.trim(alias)
          if #trimmed > 0 then table.insert(terms, trimmed) end
        end
      elseif type(aliases) == "table" then
        for _, alias in ipairs(aliases) do
          if type(alias) == "string" and #alias > 0 then
            table.insert(terms, alias)
          end
        end
      end
    end
  end
  return terms
end

-- ============ Safe position detection ============

-- Returns true if the character at startPos is in safe plain text
-- (not inside a wikilink, inline code, markdown link URL, or hashtag)
local function isSafePosition(line, startPos)
  local before = string.sub(line, 1, startPos - 1)

  -- Preceded by # means it's a hashtag, not a plain text mention
  if #before > 0 and string.sub(before, -1) == "#" then
    return false
  end

  -- Inside [[...]]
  local openCount = select(2, string.gsub(before, "%[%[", ""))
  local closeCount = select(2, string.gsub(before, "%]%]", ""))
  if openCount > closeCount then return false end

  -- Inside inline code (unpaired backtick)
  local backtickCount = select(2, string.gsub(before, "`", ""))
  if backtickCount % 2 == 1 then return false end

  -- Inside markdown link URL [text](url)
  local lastOpenParen = string.find(before, "%([^%)]*$")
  if lastOpenParen then
    local beforeParen = string.sub(before, 1, lastOpenParen - 1)
    if string.find(beforeParen, "%[[^%]]*$") then return false end
  end

  return true
end

-- Find the first safe mention of term in content, skipping frontmatter and code blocks
local function findFirstSafeMention(content, term)
  local termLower = string.lower(term)
  local inFrontmatter = false
  local frontmatterDone = false
  local inCodeBlock = false

  for line in string.gmatch(content, "([^\r\n]*)\r?\n?") do
    if not frontmatterDone and string.match(line, "^---%s*$") then
      inFrontmatter = not inFrontmatter
      if not inFrontmatter then frontmatterDone = true end
    elseif inFrontmatter then
      -- skip
    elseif string.match(line, "^```") then
      inCodeBlock = not inCodeBlock
    elseif not inCodeBlock then
      local lineLower = string.lower(line)
      local searchPos = 1
      while true do
        local s = string.find(lineLower, termLower, searchPos, true)
        if not s then break end
        if isSafePosition(line, s) then
          return line
        end
        searchPos = s + #term
      end
    end
  end
  return nil
end

-- ============ Text replacement ============

-- Replace the first safe mention of term with a wikilink
local function replaceFirstMention(content, term, targetPage)
  local lines = {}
  local inFrontmatter = false
  local frontmatterDone = false
  local inCodeBlock = false
  local replaced = false

  for line in string.gmatch(content, "([^\r\n]*)\r?\n?") do
    if not replaced then
      if not frontmatterDone and string.match(line, "^---%s*$") then
        if not inFrontmatter then
          inFrontmatter = true
        else
          inFrontmatter = false
          frontmatterDone = true
        end
        table.insert(lines, line)
      elseif inFrontmatter then
        table.insert(lines, line)
      elseif string.match(line, "^```") then
        inCodeBlock = not inCodeBlock
        table.insert(lines, line)
      elseif inCodeBlock then
        table.insert(lines, line)
      else
        local termLower = string.lower(term)
        local lineLower = string.lower(line)
        local searchPos = 1
        local newLine = line

        while true do
          local s, e = string.find(lineLower, termLower, searchPos, true)
          if not s then break end
          if isSafePosition(newLine, s) then
            local actualText = string.sub(newLine, s, e)
            local replacement
            if actualText == targetPage then
              replacement = "[[" .. targetPage .. "]]"
            else
              replacement = "[[" .. targetPage .. "|" .. actualText .. "]]"
            end
            newLine = string.sub(newLine, 1, s - 1) .. replacement .. string.sub(newLine, e + 1)
            replaced = true
            break
          end
          searchPos = e + 1
        end

        table.insert(lines, newLine)
      end
    else
      table.insert(lines, line)
    end
  end

  return table.concat(lines, "\n"), replaced
end

-- Convert a single page's first safe mention
local function linkMention(sourcePage, targetPage, term)
  local ok, content = pcall(space.readPage, sourcePage)
  if not ok or not content then
    editor.flashNotification("Failed to read page: " .. sourcePage)
    return false
  end

  local newContent, didReplace = replaceFirstMention(content, term, targetPage)
  if not didReplace then
    editor.flashNotification("No safe mention found to convert")
    return false
  end

  local writeOk = pcall(space.writePage, sourcePage, newContent)
  if not writeOk then
    editor.flashNotification("Failed to write: " .. sourcePage)
    return false
  end

  editor.flashNotification("Linked: " .. sourcePage)
  return true
end

-- Batch convert
local function linkMentions(resultList, targetPage)
  local linked = 0
  for _, r in ipairs(resultList) do
    local ok, content = pcall(space.readPage, r.id)
    if ok and content then
      local newContent, didReplace = replaceFirstMention(content, r.term, targetPage)
      if didReplace then
        local writeOk = pcall(space.writePage, r.id, newContent)
        if writeOk then linked = linked + 1 end
      end
    end
  end
  if linked > 0 then
    editor.flashNotification("Converted " .. linked .. " mentions to wikilinks")
  else
    editor.flashNotification("No mentions to convert")
  end
end

-- ============ Search ============

local function searchUnlinked(pageName, options)
  local minTermLength = options.minTermLength or 2

  local terms = {pageName}
  local pageText = editor.getText()
  for _, alias in ipairs(getAliases(pageText)) do
    table.insert(terms, alias)
  end

  local seenTerms = {}
  local validTerms = {}
  for _, term in ipairs(terms) do
    if #term >= minTermLength and not seenTerms[term] then
      seenTerms[term] = true
      table.insert(validTerms, term)
    end
  end
  if #validTerms == 0 then return {} end

  -- Already-linked pages (shown in Linked Mentions)
  local linkedSet = {}
  local linkedMentions = query[[
    from l = index.tag "link"
    where l.toPage == pageName
    select l.page
  ]]
  for _, p in ipairs(linkedMentions) do linkedSet[p] = true end

  local excludeFolders = options.excludeFolders or {}
  local seenPages = {}
  local results = {}
  local searchOk = false

  for _, term in ipairs(validTerms) do
    local ok, searchResults = pcall(function()
      return silversearch.search(term, { silent = true })
    end)

    if ok and searchResults then
      searchOk = true
      for _, r in ipairs(searchResults) do
        local rId = r.name or r.id
        if rId
          and rId ~= pageName
          and not linkedSet[rId]
          and not seenPages[rId]
          and not shouldExclude(rId, excludeFolders) then

          -- Verify with findFirstSafeMention:
          -- 1) Exclude hashtags (# prefix)
          -- 2) Exclude code blocks / frontmatter
          -- 3) Exclude text inside existing wikilinks
          local readOk, pageContent = pcall(space.readPage, rId)
          if readOk and pageContent then
            local safeLine = findFirstSafeMention(pageContent, term)
            if safeLine then
              seenPages[rId] = true
              table.insert(results, {
                id = rId,
                score = r.score or 0,
                term = term,
                excerpt = getExcerptText(r)
              })
            end
          end
        end
      end
    end
  end

  if not searchOk then return {} end

  table.sort(results, function(a, b) return a.score > b.score end)
  return results
end

-- ============ Render ============

function widgets.unlinkedMentions(pageName)
  pageName = pageName or editor.getCurrentPage()
  local options = config.get("unlinkedMentions")
  if not options or not options.enabled then return nil end

  local excludeFolders = options.excludeFolders or {}

  -- Don't render on excluded pages (Library, System, templates, etc.)
  if shouldExclude(pageName, excludeFolders) then
    return nil
  end

  local results = searchUnlinked(pageName, options)
  if #results == 0 then return nil end

  local maxResults = options.maxResults or 30
  local visible = math.min(#results, maxResults)
  local hiddenCount = #results - visible
  local defaultOpen = options.defaultOpen ~= false

  local visibleResults = {}
  local itemNodes = {}

  for i = 1, visible do
    local r = results[i]
    table.insert(visibleResults, r)

    local headChildren = {
      dom.a {
        class = "sb-unlinked-link",
        onclick = function() editor.navigate({ page = r.id }) end,
        r.id
      }
    }

    if r.term ~= pageName then
      table.insert(headChildren, dom.span {
        class = "sb-unlinked-term",
        '("' .. r.term .. '")'
      })
    end

    -- Per-item convert button
    table.insert(headChildren, dom.button {
      class = "sb-unlinked-link-one",
      title = "Convert first safe mention to wikilink",
      onclick = function() linkMention(r.id, pageName, r.term) end,
      "Link"
    })

    local bodyChildren = {
      dom.div {
        class = "sb-unlinked-item-head",
        table.unpack(headChildren)
      }
    }

    if r.excerpt and #r.excerpt > 0 then
      local snippet = string.gsub(r.excerpt, "\n", " ")
      snippet = string.gsub(snippet, "%s+", " ")
      snippet = string.trim(snippet)
      if #snippet > 120 then
        snippet = string.sub(snippet, 1, 117) .. "..."
      end
      if #snippet > 0 then
        table.insert(bodyChildren, dom.div {
          class = "sb-unlinked-excerpt",
          snippet
        })
      end
    end

    table.insert(itemNodes, dom.div {
      class = "sb-unlinked-item",
      dom.div {
        class = "sb-unlinked-item-body",
        table.unpack(bodyChildren)
      }
    })
  end

  if hiddenCount > 0 then
    table.insert(itemNodes, dom.div {
      class = "sb-unlinked-more",
      "...and " .. hiddenCount .. " more"
    })
  end

  -- Link all button
  table.insert(itemNodes, dom.div {
    class = "sb-unlinked-actions",
    dom.button {
      class = "sb-unlinked-btn",
      title = "Convert all visible mentions to wikilinks",
      onclick = function() linkMentions(visibleResults, pageName) end,
      "Link All (" .. visible .. ")"
    }
  })

  return widget.new {
    html = dom.details {
      class = "sb-unlinked",
      open = defaultOpen,
      dom.summary {
        "Unlinked Mentions ",
        dom.span { class = "sb-unlinked-count", "(" .. #results .. ")" }
      },
      dom.div {
        class = "sb-unlinked-list",
        table.unpack(itemNodes)
      }
    },
    display = "block"
  }
end

-- ============ Mount ============

event.listen {
  name = "hooks:renderBottomWidgets",
  run = function()
    local enabled = config.get("unlinkedMentions.enabled")
    if enabled ~= false then
      return widgets.unlinkedMentions()
    end
  end
}
```

## Configuration

```space-lua
config.set("unlinkedMentions", {
  enabled = true,
  maxResults = 30,
  minTermLength = 2,
  defaultOpen = true,
  excludeFolders = {"Library/", "System/", "template/", "Template/"}
})
```
