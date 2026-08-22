---
title: "如何写一篇博客文章"
date: 2026-08-22T17:13:18Z
tags: ["meta", "hugo", "blog"]
author: "COSMOSPRO"
draft: false
summary: "本站基于 Hugo + GitHub Pages，本文介绍发文流程与 frontmatter 规范。"
---

> 本文面向 blog 作者（即你），介绍怎样写一篇新文章并自动部署上线。
> 读者：自己；目的：把"发文套路"写下来，下次别再翻聊天记录。

## 1. 整体流程

```
写文章 → git push → 自动部署 → 1-2 分钟后线上可见
```

`git push origin main` 触发 `.github/workflows/hugo.yml`，Hugo 渲染 → artifact → Pages CDN。

## 2. 三种创建方式

### 方式 A：CLI（推荐，本地装 Hugo 时）

```bash
hugo new posts/2026-08-22-my-new-post.md
```

这条命令会从 `archetypes/default.md` 复制一份模板到 `content/posts/`，文件名带日期前缀，frontmatter 自动填好。

### 方式 B：手动复制

```bash
cp archetypes/default.md content/posts/2026-08-22-my-new-post.md
```

等价于方式 A，但不需要本机装 Hugo。

### 方式 C：直接在 GitHub 网页编辑

`Add file → Create new file`，路径写 `content/posts/2026-08-22-xxx.md`，把下面的模板粘进去。

## 3. frontmatter 字段

每篇文章开头必须有一段 `---` 包起来的元数据：

```yaml
---
title: "如何写一篇博客文章"            # 必填：首页卡片 + 浏览器标签
date: 2026-08-22T19:55:00Z            # 必填：发布时间，影响排序
tags: ["meta", "hugo", "blog"]        # 必填：标签数组，决定 /tags/<name>/ 页
author: "COSMOSPRO"                   # 必填：作者，显示在文章末尾 "By ..."
draft: false                          # 必填：false 才能上线（true 不会渲染）
summary: "本站基于 Hugo..."            # 可选：留空就用 <!--more--> 自动切片
---
```

四个**必填**字段是你之前定的规范：title / date / tags / author。`draft` 实际上也是必填的，否则文章不会上线。

## 4. <!--more-->：首页摘要分割线

Hugo 默认把整篇文章塞进首页。要让首页只显示摘要，在你想"切断"的位置插入：

```markdown
这里是简介（会出现在首页卡片）

<!--more-->

从这里开始是正文（只在文章页显示）
```

**规则**：
- 首页卡片 = `<!--more-->` **之前**的内容
- 文章页 = 整篇（包含分隔线）
- 没加 `<!--more-->` → 整篇都进首页（不要这样做）

## 5. archetype 模板：少写样板代码

`archetypes/default.md` 是新文章的默认模板。当前长这样：

```markdown
---
title: "{ replace .Name "-" " " | title }"
date: { .Date }
tags: []
author: "COSMOSPRO"
draft: true
summary: ""
---

一句话说明本文主题（会出现在首页卡片标题下）。frontmatter.title 已经是标题，正文里**不要**再写 `# 标题`。

<!--more-->

## 正文从这里开始
```

跑 `hugo new posts/xxx.md` 时，Hugo 会自动：
- 把 `{ replace .Name "-" " " | title }` 替换成"把文件名 `-` 换成空格、首字母大写"
- 把 `{ .Date }` 替换成当前时间
- 其它字面量原样保留

**常见改动**：每次写新文章时，把 draft 改成 false、把 tags 数组填好、把"一句话说明"换成自己的简介。

## 6. 完整示例

下面是一个**完整可用的文章文件**：

```markdown
---
title: "我的第一篇 Hugo 文章"
date: 2026-08-22T20:00:00Z
tags: ["hugo", "getting-started"]
author: "COSMOSPRO"
draft: false
summary: "在 tech_blog 上写第一篇文章的完整示范。"
---

这是简介，会出现在首页卡片里，读者看完简介决定要不要点"继续阅读"。

<!--more-->

## 第一个小节

正文从这里开始写。Markdown 全部支持：

- **粗体**、*斜体*
- [链接](https://example.com)
- 行内 `代码` 和代码块：

```python
def hello():
    print("Hello, Hugo!")
```

## 第二个小节

可以加图片、表格、引用 —— Hugo 自动渲染。

> Hugo 默认主题 ananke 是响应式的，手机上也能看。

## 7. 命名与部署

### 文件名约定

`content/posts/YYYY-MM-DD-slug.md`，日期前缀让文章按时间排序，`slug` 是 URL 片段（中文也支持）。

### 提交部署

```bash
git add content/posts/2026-08-22-my-new-post.md
git commit -m "post: 我的第一篇 Hugo 文章"
git push origin main
```

### 时间线

| 阶段 | 耗时 |
|---|---|
| git push 完成 | 几秒 |
| GitHub Actions 跑完 workflow | 30-60 秒 |
| Pages CDN 刷新 | 1-2 分钟（偶尔更长） |

## 8. 常见错误

| 现象 | 原因 | 解决 |
|---|---|---|
| 写好了但首页看不到 | `draft: true` 没改 | 改 false |
| 首页能看到但点进去 404 | 文件名没 `posts/` 前缀 | 必须在 `content/posts/` 下 |
| 卡片里文章重复出现两次 h1 | 正文里又写了 `# 标题` | 删掉，frontmatter.title 已经是标题了 |
| 整篇文章都进首页 | 没加 `<!--more-->` | 加到合适位置 |
| tag 页找不到文章 | tags 拼错或没引号 | `tags: ["a", "b"]` 必须是数组 |
| 中文标签乱码 | 极少；用英文 tags 更稳 | 推荐 tags 写英文 |

## 9. 本地预览（可选）

需要本机装 Hugo 0.160.0+ extended：

```bash
hugo server -D
# → http://localhost:1313/tech_blog/
```

`-D` 包含 draft 文章。改完 Markdown 浏览器自动刷新（live reload）。
