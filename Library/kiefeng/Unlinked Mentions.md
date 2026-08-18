---
name: Library/kiefeng/Unlinked Mentions
tags: meta/library
share.uri: "https://github.com/kiefeng/sb-unlinked-mentions/blob/main/Library/kiefeng/Unlinked Mentions.md"
share.mode: pull
---

# Unlinked Mentions Widget

```space-style
/* 折叠容器 */
.sb-unlinked {
  margin-top: 0.5rem;
}
/* 折叠标题：隐藏箭头，标题按 H1 原样渲染（与 Linked Mentions 一致） */
.sb-unlinked-summary {
  cursor: pointer;
  user-select: none;
  list-style: none;
  display: block;
  padding: 0;
}
.sb-unlinked-summary::-webkit-details-marker {
  display: none;
}
/* H1 清零默认 margin，收紧 line-height，让标题背景与框体上缘对齐 */
.sb-unlinked-summary h1 {
  margin: 0 !important;
  padding: 0 !important;
  line-height: 1.2 !important;
}
.sb-unlinked-summary:hover {
  opacity: 0.85;
}
.sb-unlinked-count {
  font-weight: normal;
  font-size: 0.5em;
  opacity: 0.55;
}
.sb-unlinked-content {
  padding: 2px 0 6px 0;
}
.sb-unlinked-item {
  padding: 7px 0;
}
.sb-unlinked-item-head {
  display: flex;
  align-items: center;
  gap: 8px;
  line-height: 1.6;
}
.sb-unlinked-link {
  color: var(--ui-link-color, var(--ui-accent-color, #58a6ff));
  font-weight: bold;
  text-decoration: none;
  cursor: pointer;
}
.sb-unlinked-link:hover {
  text-decoration: underline;
}
.sb-unlinked-term {
  opacity: 0.55;
  font-size: 0.9em;
}
.sb-unlinked-one {
  font-size: 0.85em;
  padding: 0 6px;
  cursor: pointer;
  background: none;
  border: none;
  color: var(--ui-accent-color, #888);
  opacity: 0.5;
}
.sb-unlinked-one:hover {
  opacity: 1;
  text-decoration: underline;
}
.sb-unlinked-excerpt {
  font-size: 0.9em;
  line-height: 1.5;
  opacity: 0.7;
  padding-left: 4px;
  margin-top: 3px;
}
.sb-unlinked-actions {
  padding-top: 6px;
}
.sb-unlinked-btn {
  font-size: 0.88em;
  padding: 2px 10px;
  cursor: pointer;
  background: none;
  border: 1px solid var(--ui-accent-color, #888);
  border-radius: 3px;
  color: var(--ui-accent-color, #888);
  opacity: 0.6;
}
.sb-unlinked-btn:hover {
  opacity: 1;
}
.sb-unlinked-more {
  font-size: 0.85em;
  opacity: 0.5;
  padding: 2px 0;
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
    contextLen = schema.number(),
    excludeFolders = schema.array("string"),
  }
})

config.set("unlinkedMentions", {
  enabled = true,
  maxResults = 30,
  minTermLength = 2,
  defaultOpen = true,
  contextLen = 100,
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
  if #before > 0 and string.sub(before, -1) == "#" then return false end
  local openCount = select(2, string.gsub(before, "%[%[", ""))
  local closeCount = select(2, string.gsub(before, "%]%]", ""))
  if openCount > closeCount then return false end
  local backtickCount = select(2, string.gsub(before, "`", ""))
  if backtickCount % 2 == 1 then return false end
  local lastOpenParen = string.find(before, "%([^%)]*$")
  if lastOpenParen then
    local beforeParen = string.sub(before, 1, lastOpenParen - 1)
    if string.find(beforeParen, "%[[^%]]*$") then return false end
  end
  return true
end

local function findFirstSafeMention(content, term)
  local termLower = string.lower(term)
  local inFrontmatter = false
  local frontmatterDone = false
  local inCodeBlock = false
  for line in string.gmatch(content, "([^\r\n]*)\r?\n?") do
    if not frontmatterDone and string.match(line, "^---%s*$") then
      inFrontmatter = not inFrontmatter
      if not inFrontmatter then frontmatterDone = true end
    elseif not inFrontmatter and string.match(line, "^```") then
      inCodeBlock = not inCodeBlock
    elseif not inFrontmatter and not inCodeBlock then
      local lineLower = string.lower(line)
      local searchPos = 1
      while true do
        local s, e = string.find(lineLower, termLower, searchPos, true)
        if not s then break end
        if isSafePosition(line, s) then return line, s, e end
        searchPos = e + 1
      end
    end
  end
  return nil
end

-- 递归提取语法树中的纯文本（官方解析器保证，不依赖正则）
local function treeToText(node)
  if not node then return "" end
  -- 跳过纯标记节点：[[ ]] 括号、标题 #、引用 > 等，只保留有意义文本
  if node.type == "WikiLinkMark"
    or node.type == "HeaderMark"
    or node.type == "QuoteMark"
    or node.type == "ListItemMark"
    or node.type == "EmphasisMark"
    or node.type == "CodeMark"
    or node.type == "LinkMark"
    or node.type == "URLMark"
    or node.type == "TaskMark"
    or node.type == "CheckboxMark" then
    return ""
  end
  if node.type == nil then
    -- 文本节点
    return node.text or ""
  end
  if node.children then
    local parts = {}
    for _, child in ipairs(node.children) do
      table.insert(parts, treeToText(child))
    end
    return table.concat(parts)
  end
  return ""
end

local function extractSnippet(line, startPos, endPos, contextLen)
  contextLen = contextLen or 100
  local lineStart = math.max(1, startPos - contextLen)
  local lineEnd = math.min(#line, endPos + contextLen)
  local snippet = string.sub(line, lineStart, lineEnd)
  if lineStart > 1 then snippet = "..." .. snippet end
  if lineEnd < #line then snippet = snippet .. "..." end
  -- 用官方 markdown 解析器把 snippet 转纯文本：
  -- # 标题、**粗体**、[[链接]]、[text](url) 等全部剥掉，只留文字
  local ok, tree = pcall(markdown.parseMarkdown, snippet)
  if ok and tree then
    snippet = treeToText(tree)
  end
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
        if not inFrontmatter then inFrontmatter = true
        else inFrontmatter = false; frontmatterDone = true end
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
    -- 区分：页面里有没有这个提及？有但都在不安全位置？
    local hasMention = string.find(string.lower(content), string.lower(term), 1, true) ~= nil
    if hasMention then
      editor.flashNotification("Mention found but only in unsafe locations (code/link/frontmatter)")
    else
      editor.flashNotification("No mention of \"" .. term .. "\" in page")
    end
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

local function searchUnlinked(pageName, options, pageText)
  local minTermLength = options.minTermLength or 2
  local terms = {pageName}
  -- pageText 参数：widget 渲染时传 nil（用当前编辑器文本），
  -- 全库命令时传 space.readPage(pageName) 的文本（保证 aliases 属于被遍历的页面）
  if not pageText then
    pageText = editor.getText()
  end
  for _, alias in ipairs(getAliases(pageText)) do table.insert(terms, alias) end

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
        if rId and rId ~= pageName and not linkedSet[rId]
          and not seenPages[rId] and not shouldExclude(rId, excludeFolders) then
          local readOk, pageContent = pcall(space.readPage, rId)
          if readOk and pageContent then
            local line, s, e = findFirstSafeMention(pageContent, term)
            if line then
              seenPages[rId] = true
              table.insert(results, {
                id = rId,
                score = r.score or 0,
                term = term,
                snippet = extractSnippet(line, s, e, options.contextLen)
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
  if shouldExclude(pageName, excludeFolders) then return nil end

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

    table.insert(headChildren, dom.button {
      class = "sb-unlinked-one",
      title = "Convert first safe mention to wikilink",
      onclick = function() linkMention(r.id, pageName, r.term) end,
      "Link"
    })

    local itemChildren = {
      dom.div {
        class = "sb-unlinked-item-head",
        table.unpack(headChildren)
      }
    }

    if r.snippet and #r.snippet > 0 then
      table.insert(itemChildren, dom.div {
        class = "sb-unlinked-excerpt",
        __rawText = r.snippet
      })
    end

    table.insert(itemNodes, dom.div {
      class = "sb-unlinked-item",
      table.unpack(itemChildren)
    })
  end

  if hiddenCount > 0 then
    table.insert(itemNodes, dom.div {
      class = "sb-unlinked-more",
      "...and " .. hiddenCount .. " more"
    })
  end

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
        class = "sb-unlinked-summary",
        "# Unlinked Mentions (" .. tostring(#results) .. ")",
      },
      dom.div {
        class = "sb-unlinked-content",
        table.unpack(itemNodes)
      }
    },
    display = "block"
  }
end

-- ============ Commands ============

-- Convert ALL unlinked mentions across the ENTIRE space
-- (every page x every mention), with a confirmation prompt
pcall(function()
  command.define {
    name = "Unlinked Mentions: Link All (Full Space)",
    description = "Convert EVERY unlinked mention in the whole space to a wikilink (heavy operation, modifies many pages)",
    run = function()
      local options = config.get("unlinkedMentions")
      if not options or not options.enabled then return end

      -- List all pages in the space
      local ok, allPages = pcall(space.listPages)
      if not ok or not allPages then
        editor.flashNotification("Failed to list pages")
        return
      end

      -- Confirm: this is a heavy, space-wide operation
      local confirmed = editor.confirm(
        "Convert EVERY unlinked mention in the whole space to wikilinks?\n\n"
        .. "This will scan " .. #allPages .. " page(s) and modify many of them.\n"
        .. "It may take a while. Use with care."
      )
      if not confirmed then
        editor.flashNotification("Cancelled")
        return
      end

      local excludeFolders = options.excludeFolders or {}
      local mentionsLinked = 0
      local processed = 0
      local total = #allPages

      -- For each page in the space, find pages that mention it without a link
      for _, pageMeta in ipairs(allPages) do
        local pageName = pageMeta.name or pageMeta
        if type(pageName) == "string"
          and not shouldExclude(pageName, excludeFolders) then

          -- 读取该页面自身的文本，保证 aliases 属于被遍历的页面
          -- （读取失败时 selfText 为 nil，searchUnlinked 会回退到当前编辑器文本，行为安全）
          local selfOk, selfText = pcall(space.readPage, pageName)
          if not selfOk then selfText = nil end
          local results = searchUnlinked(pageName, options, selfText)
          for _, r in ipairs(results) do
            local readOk, content = pcall(space.readPage, r.id)
            if readOk and content then
              local newContent, didReplace = replaceFirstMention(content, r.term, pageName)
              if didReplace then
                local writeOk = pcall(space.writePage, r.id, newContent)
                if writeOk then
                  mentionsLinked = mentionsLinked + 1
                end
              end
            end
          end
        end

        -- 每 50 页提示一次进度（轻量反馈，避免长时间无响应感）
        processed = processed + 1
        if processed % 50 == 0 then
          editor.flashNotification("Full-space linking: " .. processed .. "/" .. total .. " pages scanned")
        end
      end

      if mentionsLinked > 0 then
        editor.flashNotification("Linked " .. mentionsLinked .. " mentions across " .. total .. " pages")
      else
        editor.flashNotification("No mentions could be converted")
      end
    end
  }
end)

-- Convert ALL unlinked mentions on the CURRENT page (not limited by maxResults),
-- with a confirmation prompt
pcall(function()
  command.define {
    name = "Unlinked Mentions: Link All (This Page)",
    description = "Convert every unlinked mention of the current page to a wikilink (not limited to maxResults)",
    run = function()
      local pageName = editor.getCurrentPage()
      local options = config.get("unlinkedMentions")
      if not options or not options.enabled then return end

      local results = searchUnlinked(pageName, options)
      if #results == 0 then
        editor.flashNotification("No unlinked mentions found")
        return
      end

      local confirmed = editor.confirm(
        "Convert ALL " .. #results .. " unlinked mentions of this page to wikilinks?\n\n"
        .. "This will modify " .. #results .. " page(s)."
      )
      if not confirmed then
        editor.flashNotification("Cancelled")
        return
      end

      local linked = 0
      for _, r in ipairs(results) do
        local ok, content = pcall(space.readPage, r.id)
        if ok and content then
          local newContent, didReplace = replaceFirstMention(content, r.term, pageName)
          if didReplace then
            pcall(space.writePage, r.id, newContent)
            linked = linked + 1
          end
        end
      end
      if linked > 0 then
        editor.flashNotification("Linked " .. linked .. " of " .. #results .. " mentions")
      else
        editor.flashNotification("No mentions could be converted")
      end
    end
  }
end)

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

```lua
config.set("unlinkedMentions", {
  enabled = true,
  maxResults = 30,
  minTermLength = 2,
  defaultOpen = true,
  excludeFolders = {"Library/", "System/", "template/", "Template/"}
})
```
