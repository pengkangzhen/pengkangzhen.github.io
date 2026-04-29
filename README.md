# 我的博客

使用 [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题搭建的个人博客。

## 本地运行

```bash
# 安装 Hugo
brew install hugo

# 克隆仓库（含主题子模块）
git clone --recurse-submodules https://github.com/pengkangzhen/pengkangzhen.github.io.git
cd pengkangzhen.github.io

# 本地预览
hugo server -D
```

## 写文章

```bash
# 新建文章
hugo new content posts/my-new-post.md

# 编辑 content/posts/my-new-post.md
# 推送到 GitHub 即自动部署
```

## 部署

推送到 `main` 分支后，GitHub Actions 会自动构建并部署到 GitHub Pages。
