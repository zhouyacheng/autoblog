---
title: "为什么用 Hugo + GitHub Pages 写博客"
date: 2026-10-26T21:25:00+08:00
tags: [hugo, github-pages, workflow]
author: zyc
---

博客这件事，卡点从来不是"写"，而是"发"：环境要装、构建要等、发布要点好几次。所以这次的最小目标很简单——**改完 Markdown 提个 PR，合并之后线上就是新的**。

## 选型

- **Hugo**：单二进制、构建快，几千篇文章也是秒级。
- **GitHub Pages**：不用服务器，和仓库天然一体，push 即发布。
- **PaperMod**：主题成熟，自带搜索、暗色模式，不折腾样式。

## 发布流程

1. 新文章落在 `content/posts/` 下，文件名即 URL slug。
2. 从 `main` 切分支，提交 PR。
3. 评审通过合并到 `main`。
4. GitHub Actions 自动构建并部署，几十秒后线上可见。

frontmatter 只保留四个字段：

```yaml
---
title: "文章标题"
date: 2026-10-26T21:25:00+08:00
tags: [hugo, workflow]
author: zyc
---
```

## 接下来

先把写作流跑顺，再考虑 RSS、标签页优化和评论系统。工具够用就行，别让它变成新的负担。
