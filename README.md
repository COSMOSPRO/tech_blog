# tech_blog

技术博客，Hugo + GitHub Pages。

- 仓库：https://github.com/COSMOSPRO/tech_blog
- 站点：https://COSMOSPRO.github.io/tech_blog/
- 主题：[ananke](https://github.com/theNewDynamic/gohugo-theme-ananke)
- Hugo 版本：0.160.0（extended）

## 目录

```
archetypes/default.md      # 新文章 frontmatter 模板（title/date/tags/author）
content/posts/             # 所有文章放这里
themes/ananke/             # git submodule
.github/workflows/hugo.yml # push 到 main 自动部署
hugo.toml                  # 站点配置
```

## 发文章流程

1. 在 `content/posts/` 新建 `your-slug.md`，frontmatter 至少包含 title/date/tags/author
2. 提交 PR 到 main（合并即部署）
3. workflow build 通过后几分钟内，文章出现在 https://COSMOSPRO.github.io/tech_blog/posts/your-slug/

## 本地预览

需要 Hugo ≥ 0.160.0 extended：

```bash
hugo server -D
# → http://localhost:1313/tech_blog/
```
