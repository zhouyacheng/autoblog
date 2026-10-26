# Autoblog

Hugo + GitHub Pages 博客。文章在 `content/posts/`，frontmatter 字段：`title` / `date` / `tags` / `author`。

## 发布流程

```bash
# 1. 新建文章（自动套用 archetypes/default.md 的四字段模板）
hugo new content/posts/my-post.md

# 2. 本地预览（含草稿）
hugo server -D

# 3. 提交 PR
git switch -c post/my-post
git add content/posts/my-post.md
git commit -m "content: add my-post"
git push -u origin post/my-post
```

合并到 `main` 后，`.github/workflows/hugo.yml` 自动构建并部署到 GitHub Pages。

## 本地环境

- Hugo extended ≥ 0.146（`brew install hugo`）
- 主题以 git submodule 形式引入，克隆时用 `git clone --recurse-submodules` 或 `git submodule update --init`
