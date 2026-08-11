---
name: Library/kiefeng/Unlinked Mentions
tags: meta/library
share.uri: "ghr:kiefeng/sb-unlinked-mentions@main/Library/kiefeng/Unlinked Mentions.md"
share.mode: pull
---

# Unlinked Mentions Widget

```space-style
.sb-unlinked {
  margin-top: 1rem;
  border-top: 1px solid var(--ui-accent-color, #888);
  padding-top: 0.5rem;
}
.sb-unlinked-summary {
  cursor: pointer;
  font-weight: bold;
  font-size: 1.1em;
  user-select: none;
  padding: 4px 0;
  display: flex;
  align-items: center;
  gap: 8px;
}
.sb-unlinked-count {
  font-weight: normal;
  font-size: 0.85em;
  opacity: 0.6;
}
.sb-unlinked-list {
  padding-top: 0.4rem;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.sb-unlinked-item {
  display: flex;
  align-items: flex-start;
  gap: 6px;
  font-size: 0.95em;
  padding: 2px 0;
}
.sb-unlinked-item-body {
  flex: 1;
  min-width: 0;
}
.sb-unlinked-item-head {
  display: flex;
  align-items: center;
  gap: 6px;
}
.sb-unlinked-link-one {
  flex-shrink: 0;
  font-size: 0.8em;
  padding: 1px 8px;
  border: 1px solid var(--ui-accent-color, #888);
  border-radius: 3px;
  background: transparent;
  color: var(--ui-accent-color, #888);
  cursor: pointer;
  opacity: 0.4;
  transition: opacity 0.15s;
}
.sb-unlinked-link-one:hover {
  opacity: 1;
}
.sb-unlinked-link {
  cursor: pointer;
  color: var(--ui-link-color, var(--ui-accent-color, #58a6ff));
  text-decoration: none;
}
.sb-unlinked-link:hover {
  text-decoration: underline;
}
.sb-unlinked-term {
  font-size: 0.85em;
  opacity: 0.55;
}
.sb-unlinked-excerpt {
  font-size: 0.85em;
  opacity: 0.5;
  border-left: 2px solid var(--ui-accent-color, #888);
  padding: 1px 0 1px 8px;
  margin-top: 2px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
.sb-unlinked-actions {
  display: flex;
  gap: 8px;
  padding-top: 8px;
  margin-top: 4px;
}
.sb-unlinked-btn {
  font-size: 0.85em;
  padding: 3px 12px;
  border: 1px solid var(--ui-accent-color, #888);
  border-radius: 4px;
  background: transparent;
  color: var(--ui-accent-color, #888);
  cursor: pointer;
  opacity: 0.7;
  transition: opacity 0.15s;
}
.sb-unlinked-btn:hover {
  opacity: 1;
}
.sb-unlinked-btn-primary {
  background: var(--ui-accent-color, #888);
  color: var(--ui-background-color, #fff);
  border-color: var(--ui-accent-color, #888);
  opacity: 0.85;
}
.sb-unlinked-btn-primary:hover {
  opacity: 1;
}
.sb-unlinked-more {
  font-size: 0.85em;
  opacity: 0.5;
  padding-top: 4px;
}
```

```space-lua
-- priority: 10
widgets = widgets or {}

-- ============ 配置 ============
config.define("std.widgets.unlinkedMentions", {
  type = "object",
  properties = {
    enabled = schema.boolean(),
    maxResults = schema.number(),
    minTermLength = schema.number(),
    defaultOpen = schema.boolean(),
    excludeFolders = schema.array("string"),
  }
})

config.set("std.widgets.unlinkedMentions", {
  enabled = true,
  maxResults = 30,
  minTermLength = 2,
  defaultOpen = true,
  excludeFolders = {"Library/", "System/", "template/", "Template/"}
})

-- ============ 基础工具 ============

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

-- ============ 安全位置检测 ============

-- 判断某行中 startPos 位置是否是安全的纯文本区域
-- 安全 = 不在双链内、不在行内代码内、不在 Markdown 链接 URL 内、不是标签（# 前缀）
local function isSafePosition(line, startPos)
  local before = string.sub(line, 1, startPos - 1)

  -- 前面紧跟 # 说明是标签，不算纯文本提及
  if #before > 0 and string.sub(before, -1) == "#" then
    return false
  end

  -- 在 [[...]] 内
  local openCount = select(2, string.gsub(before, "%[%[", ""))
  local closeCount = select(2, string.gsub(before, "%]%]", ""))
  if openCount > closeCount then return false end

  -- 在行内代码内（反引号未配对）
  local backtickCount = select(2, string.gsub(before, "`", ""))
  if backtickCount % 2 == 1 then return false end

  -- 在 Markdown 链接的 URL 部分 [text](url)
  local lastOpenParen = string.find(before, "%([^%)]*$")
  if lastOpenParen then
    local beforeParen = string.sub(before, 1, lastOpenParen - 1)
    if string.find(beforeParen, "%[[^%]]*$") then return false end
  end

  return true
end

-- 在全文中查找第一处安全的 term 出现，返回 (lineText, lineNum, startPos, endPos)
-- 跳过 frontmatter 和代码块
local function findFirstSafeMention(content, term)
  local termLower = string.lower(term)
  local inFrontmatter = false
  local frontmatterDone = false
  local inCodeBlock = false
  local lineNum = 0

  for line in string.gmatch(content, "([^\r\n]*)\r?\n?") do
    lineNum = lineNum + 1

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
          return line, lineNum, s, e
        end
        searchPos = e + 1
      end
    end
  end
  return nil
end

-- ============ 文本替换 ============

-- 将 content 中第一处安全的 term 替换为双链
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

-- 对单个页面执行转正
local function linkMention(sourcePage, targetPage, term)
  local ok, content = pcall(space.readPage, sourcePage)
  if not ok or not content then
    editor.flashNotification("无法读取页面: " .. sourcePage)
    return false
  end

  local newContent, didReplace = replaceFirstMention(content, term, targetPage)
  if not didReplace then
    editor.flashNotification("未找到可安全替换的提及")
    return false
  end

  local writeOk = pcall(space.writePage, sourcePage, newContent)
  if not writeOk then
    editor.flashNotification("写入失败: " .. sourcePage)
    return false
  end

  editor.flashNotification("已链接: " .. sourcePage)
  return true
end

-- 批量转正
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
    editor.flashNotification("已将 " .. linked .. " 处提及转为双链")
  else
    editor.flashNotification("没有可转换的提及")
  end
end

-- ============ 搜索逻辑 ============

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

  -- 已链接的页面集合
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

          -- 用 findFirstSafeMention 验证：
          -- 1) 排除标签（# 前缀）
          -- 2) 排除代码块/frontmatter
          -- 3) 排除已有双链内
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

-- ============ 渲染 ============

function widgets.unlinkedMentions(pageName)
  pageName = pageName or editor.getCurrentPage()
  local options = config.get("std.widgets.unlinkedMentions")
  if not options or not options.enabled then return nil end

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

    -- 单条转正按钮
    table.insert(headChildren, dom.button {
      class = "sb-unlinked-link-one",
      title = "将此页第一处提及转为双链",
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
      "...还有 " .. hiddenCount .. " 条"
    })
  end

  -- 一键转全部
  table.insert(itemNodes, dom.div {
    class = "sb-unlinked-actions",
    dom.button {
      class = "sb-unlinked-btn sb-unlinked-btn-primary",
      title = "将所有可见结果的第一处提及转为双链",
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
        "Unlinked Mentions",
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

-- ============ 挂载 ============

event.listen {
  name = "hooks:renderBottomWidgets",
  run = function()
    local enabled = config.get("std.widgets.unlinkedMentions.enabled")
    if enabled ~= false then
      return widgets.unlinkedMentions()
    end
  end
}
```

## 配置

```space-lua
config.set("std.widgets.unlinkedMentions", {
  enabled = true,
  maxResults = 30,
  minTermLength = 2,
  defaultOpen = true,
  excludeFolders = {"Library/", "System/", "template/", "Template/"}
})
```

## 功能说明

- **折叠**：点击标题栏展开/收起
- **单条转换**：每条结果右侧的 Link 按钮只转该页
- **一键全转**：底部 Link All 按钮转换所有可见结果
- **标签排除**：`#标签` 中的页面名不会被识别为未链接提及，也不会被转换
- **安全替换**：跳过 frontmatter、代码块、行内代码、已有双链、Markdown 链接 URL
- **别名支持**：命中别名时替换为 `[[真实页面名|别名原文]]`
