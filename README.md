# tech_blog

技术博客，Hugo + GitHub Pages。

- 仓库：https://github.com/COSMOSPRO/tech_blog
- 站点：https://COSMOSPRO.github.io/tech_blog/
- 主题：[ananke](https://github.com/theNewDynamic/gohugo-theme-ananke)
- Hugo 版本：0.160.0（extended）

## 目录

```
archetypes/default.md      # 新文章 frontmatter 模板（title/date/tags/author/draft/summary）
content/posts/             # 所有文章放这里
themes/ananke/             # git submodule
.github/workflows/hugo.yml # push 到 main 自动部署
hugo.toml                  # 站点配置（含 mainSections / ananke.show_recent_posts）
```

## 发文章流程

1. 在 `content/posts/` 新建 `YYYY-MM-DD-your-slug.md`
2. frontmatter 至少包含 title / date / tags / author / draft（draft: false 才能上线）
3. 用 `<!--more-->` 把"简介"和"正文"分开，简介进首页卡片
4. 提交并 push 到 main，workflow 自动 build + deploy
5. 1-2 分钟后 CDN 刷新，文章出现在首页和文章页

完整规范（字段含义、`<!--more-->` 行为、archetype 模板、常见错误）见：

📖 **[如何写一篇博客文章](https://cosmospro.github.io/tech_blog/posts/2026-08-22-how-to-write-a-post/)**

## 本地预览

需要 Hugo ≥ 0.160.0 extended：

```bash
hugo server -D
# → http://localhost:1313/tech_blog/
```

## 文档索引

- [如何写一篇博客文章](https://cosmospro.github.io/tech_blog/posts/2026-08-22-how-to-write-a-post/) — 发文规范
- [GitHub SSH Key 备份与恢复方案](https://cosmospro.github.io/tech_blog/posts/2026-08-22-ssh-key-backup-restore/) — SSH key 备份/恢复全流程
