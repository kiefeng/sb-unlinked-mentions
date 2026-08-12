---
name: Library/kiefeng/Unlinked Mentions
tags: meta/library
share.uri: "ghr:kiefeng/sb-unlinked-mentions@main/Library/kiefeng/Unlinked Mentions.md"
share.mode: pull
---

# Unlinked Mentions Widget

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

local function isSafePosition(line, startPos)
  local before = string.sub(line, 1, startPos - 1)

  -- Preceded by # means hashtag
  if #before > 0 and string.sub(before, -1) == "#" then
    return false
  end

  -- Inside [[...]]
  local openCount = select(2, string.gsub(before, "%[%[", ""))
  local closeCount = select(2, string.gsub(before, "%]%]", ""))
  if openCount > closeCount then return false end

  -- Inside inline code
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

-- Find the first safe mention line and return (lineText, startPos, endPos)
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
        local s, e = string.find(lineLower, termLower, searchPos, true)
        if not s then break end
        if isSafePosition(line, s) then
          return line, s, e
        end
        searchPos = e + 1
      end
    end
  end
  return nil
end

-- Extract context snippet around the matched term
local function extractSnippet(line, startPos, endPos, contextLen)
  contextLen = contextLen or 100
  local lineStart = math.max(1, startPos - contextLen)
  local lineEnd = math.min(#line, endPos + contextLen)
  local snippet = string.sub(line, lineStart, lineEnd)
  if lineStart > 1 then snippet = "..." .. snippet end
  if lineEnd < #line then snippet = snippet .. "..." end
  return snippet
end

-- ============ Text replacement ============

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

local function linkMention(sourcePage, targetPage, term)
  local ok, content = pcall(space.readPage, sourcePage)
  if not ok or not content then
    editor.flashNotification("Failed to read page: " .. sourcePage)
    return
  end

  local newContent, didReplace = replaceFirstMention(content, term, targetPage)
  if not didReplace then
    editor.flashNotification("No safe mention found to convert")
    return
  end

  pcall(space.writePage, sourcePage, newContent)
  editor.flashNotification("Linked: " .. sourcePage)
end

local function linkMentions(resultList, targetPage)
  local linked = 0
  for _, r in ipairs(resultList) do
    local ok, content = pcall(space.readPage, r.id)
    if ok and content then
      local newContent, didReplace = replaceFirstMention(content, r.term, targetPage)
      if didReplace then
        pcall(space.writePage, r.id, newContent)
        linked = linked + 1
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

          local readOk, pageContent = pcall(space.readPage, rId)
          if readOk and pageContent then
            local line, s, e = findFirstSafeMention(pageContent, term)
            if line then
              seenPages[rId] = true
              table.insert(results, {
                id = rId,
                score = r.score or 0,
                term = term,
                snippet = extractSnippet(line, s, e)
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

    -- Title row: **[[Page Name]]** (alias)  Link
    local titleChildren = {
      dom.strong {
        dom.a {
          onclick = function() editor.navigate({ page = r.id }) end,
          style = "cursor: pointer; color: var(--ui-link-color, var(--ui-accent-color)); text-decoration: none;",
          r.id
        }
      }
    }

    if r.term ~= pageName then
      table.insert(titleChildren, ' ("' .. r.term .. '")')
    end

    table.insert(titleChildren, "  ")
    table.insert(titleChildren, dom.a {
      onclick = function() linkMention(r.id, pageName, r.term) end,
      style = "font-size: 0.85em; opacity: 0.5; cursor: pointer; color: var(--ui-link-color, var(--ui-accent-color)); text-decoration: none;",
      "Link"
    })

    table.insert(itemNodes, dom.p(titleChildren))

    -- Snippet paragraph
    if r.snippet and #r.snippet > 0 then
      table.insert(itemNodes, dom.p {
        style = "opacity: 0.6; font-size: 0.9em; margin: 0 0 0.5em 0;",
        r.snippet
      })
    end
  end

  if hiddenCount > 0 then
    table.insert(itemNodes, dom.p {
      style = "opacity: 0.5; font-size: 0.9em;",
      "...and " .. hiddenCount .. " more"
    })
  end

  -- Link All
  table.insert(itemNodes, dom.p {
    dom.a {
      onclick = function() linkMentions(visibleResults, pageName) end,
      style = "font-size: 0.9em; opacity: 0.6; cursor: pointer; color: var(--ui-link-color, var(--ui-accent-color)); text-decoration: none;",
      "Link All (" .. visible .. ")"
    }
  })

  return widget.new {
    html = dom.details {
      open = defaultOpen,
      dom.summary {
        style = "cursor: pointer; font-weight: bold; font-size: 1.1em; margin-bottom: 0.3em;",
        "Unlinked Mentions (", tostring(#results), ")"
      },
      table.unpack(itemNodes)
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
