# yangkushu.github.io

这是一个基于 Hugo 的个人文章站点，使用 `meme` 主题，并通过 GitHub Pages 发布。

## 项目结构

- `hugo.toml`：站点配置。
- `content/posts/`：文章目录。
- `archetypes/default.md`：新文章模板，默认生成草稿。
- `themes/meme/`：Hugo 主题，以 Git submodule 管理。
- `.github/workflows/hugo.yml`：GitHub Pages 自动部署流程。

## 写文章

创建新文章：

```bash
hugo new content/posts/my-first-post.md
```

新文章默认包含：

```toml
draft = true
```

保持 `draft = true` 时，文章是草稿，不会被正式构建发布。

本地预览草稿：

```bash
hugo server -D
```

本地预览正式发布效果：

```bash
hugo server
```

## 发布

推送到 `master` 后，GitHub Actions 会自动构建并发布到 GitHub Pages。

生产构建不会带 `-D` 参数，因此 `draft = true` 的文章不会发布到网站。

注意：如果仓库是公开的，草稿文件即使不发布到网站，推送后仍可能在 GitHub 仓库源码中被看到。

## 初始化主题

首次克隆仓库后，需要初始化主题 submodule：

```bash
git submodule update --init --recursive
```

## 参考

- https://jianzhnie.github.io/post/hugo_site/
