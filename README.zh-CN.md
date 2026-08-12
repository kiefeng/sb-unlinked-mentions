# SilverBullet Unlinked Mentions（未链接提及）

一个 [SilverBullet](https://silverbullet.md) 插件：在页面底部显示**未链接提及**（在其他页面中以纯文本出现、但没有 `[[双链]]` 指向当前页面的引用），类似 Obsidian 的 Unlinked Mentions 面板，并支持一键转换为真正的双链。

> **English version**: [README.md](README.md) · **更新日志**: [CHANGELOG.md](CHANGELOG.md)

## 功能

- 🔍 **全文搜索**：基于 [Silversearch](https://github.com/MrMugame/silversearch)
- 🎯 **精确子串验证**：杜绝模糊搜索误报。Silversearch 默认模糊匹配，搜 "Journal/2026-08-11" 可能匹配到 "Journal/2026-08-07"（共享 token），插件对每个结果都会二次核对实际文本
- 🔗 **单条转正**：每条结果右侧的 `Link` 按钮，将该页第一处安全提及转为 `[[双链]]`
- ⚡ **一键全转**：底部 `Link All` 按钮批量转换所有可见结果
- 🚫 **标签排除**：`#标签` 中的提及不会被识别为未链接提及，也不会被转换
- 🛡️ **安全替换**：自动跳过 frontmatter、代码块、行内代码、已有 `[[双链]]`、Markdown 链接 URL
- 🏷️ **别名支持**：读取 frontmatter 的 `aliases`，命中别名时转成 `[[真实页面名|别名原文]]` 保留原文字
- 📂 **可折叠**：点击标题展开/收起（隐藏箭头，与 Linked Mentions 视觉一致）
- 🌏 **中文搜索**：配合 [中文分词器](https://github.com/LelouchHe/silversearch-chinese-tokenizer) 效果最佳
- 🪶 **零构建**：纯 Space Lua，一个 `.md` 文件即可，无需编译
- 🔤 **纯文本摘要**：上下文片段经官方 markdown 解析器转纯文本，原文中的 `# 标题`、`**加粗**`、`[[链接]]` 不会泄漏到预览里

## 环境要求

- SilverBullet v2.x
- [Silversearch](https://github.com/MrMugame/silversearch)（全文搜索库，先通过 Configuration Manager 安装）

中文笔记空间建议额外安装：
- [silversearch-chinese-tokenizer](https://github.com/LelouchHe/silversearch-chinese-tokenizer)（jieba 中文分词器）

## 安装

### 方式一：手动安装（最简单）

1. 从仓库下载 [`Unlinked Mentions.md`](https://github.com/kiefeng/sb-unlinked-mentions/blob/main/Library/kiefeng/Unlinked%20Mentions.md)
2. 放入你的 SilverBullet 空间任意位置（如 `Library/Unlinked Mentions.md` 或任何你喜欢的目录）
3. 如果你放的位置不是 `Library/kiefeng/Unlinked Mentions.md`，把文件 frontmatter 里的 `name` 字段改成实际路径
4. 刷新 SilverBullet（Ctrl+Shift+R）

有未链接提及的页面底部会自动出现该区块。

### 方式二：Library Manager

如果使用 [Configuration Manager](https://silverbullet.md/Library%20Manager)，把本仓库添加为库来源后安装即可，文件会自动放到 `Library/kiefeng/` 下。

## 配置

在 `CONFIG` 页面中可自定义。**以下所有选项都可以直接改——无需动插件代码**：

```space-lua
config.set("unlinkedMentions", {
  enabled = true,          -- 设为 false 完全禁用
  maxResults = 30,         -- 最多显示条数（想看更多可改成 50 等）
  minTermLength = 2,       -- 最短搜索词长度（单字不搜，避免噪声）
  defaultOpen = true,      -- 默认展开（false 则默认折叠）
  contextLen = 100,        -- 每条提及前后显示的上下文字符数
  excludeFolders = {       -- 排除的目录 —— 可以把自己的目录加进来
    "Library/",            --   例如 "私密笔记/", "工作/归档/", "Notes/Inbox/"
    "System/",
    "template/",
    "Template/"
  }
})
```

**`excludeFolders` 双向生效：**
- 排除目录里的页面**不会出现**在其他页面的提及列表里
- 当前页面属于排除目录时，widget **不渲染**
- **把自己的目录加进列表**，即可让这些目录完全不被提及搜索发现

也可以单独改某一项，不用重写整个配置：

```space-lua
-- 显示更多结果
config.set("unlinkedMentions.maxResults", 50)
-- 默认折叠
config.set("unlinkedMentions.defaultOpen", false)
-- 禁用插件
config.set("unlinkedMentions.enabled", false)
```

## 工作原理

1. 收集搜索词：当前页面名 + frontmatter 中的 `aliases`
2. 用 Silversearch 逐个搜索
3. 过滤掉：当前页自身、已有双链的页面、被排除的目录
4. 对每个结果做**精确子串验证**（检查摘要或全文是否真的包含搜索词），消除模糊匹配误报
5. 按相关度排序，在 Linked Mentions 下方渲染可折叠区块

## 转换安全性

`Link` 按钮只替换每页**第一处安全提及**，自动跳过：

- YAML frontmatter（两个 `---` 之间）
- 围栏代码块（```` ``` ````）
- 行内代码（`` `code` ``）
- 已在 `[[双链]]` 内的文本
- Markdown 链接的 URL 部分 `[text](url)`

避免破坏代码示例、已有链接，或过度链接。

## 命令

四种转正入口，范围不同：

| 入口 | 范围 | 确认 |
|---|---|---|
| 每条结果右侧 **Link** 按钮 | 单个页面的一处提及 | 无 |
| widget 底部 **Link All** 按钮 | 当前页前 `maxResults`（默认 30）条 | 无 |
| 命令 `Unlinked Mentions: Link All (This Page)` | 当前页**全部**提及（不限 30 条） | 有 |
| 命令 `Unlinked Mentions: Link All (Full Space)` | **全库**所有页面的所有提及 | 有 + 进度提示 |

命令在命令面板运行（`Ctrl-k` / `Cmd-k`）。全库命令会扫描每个页面，每 50 页提示一次进度——请谨慎使用。

## 已知限制

- 纯文本匹配，代码块/frontmatter 之外的提及（如模板内容）可能被算入，后续可加 ParseTree 过滤
- 无缓存，切换页面时重新搜索，大空间可能有轻微延迟
- 每次转换只处理每页第一处提及，避免过度链接（这正是设计意图）

## 许可证

MIT
