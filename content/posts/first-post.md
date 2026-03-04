---
title: "GitHub+Hugo 博客搭建记录"
date: 2026-03-04
draft: false
tags: ["Linux", "WSL2", "环境搭建"]
---

## 环境准备

### 安装 Git

PowerShell 中运行：

```powershell
winget install Git.Git
```

安装完成后重启终端，配置身份信息：

```powershell
git config --global user.name "名字"
git config --global user.email "邮箱"
```

验证安装：

```powershell
git --version
```

### 安装 Hugo

推荐使用 winget 安装 Hugo Extended 版本（Extended 版支持 SCSS，很多主题需要）：

```powershell
winget install Hugo.Hugo.Extended
```

> **踩坑记录：** 检索msstore源然后报错了
>
> **我的解决方式：手动下载**：去 Hugo 的 [GitHub Releases](https://github.com/gohugoio/hugo/releases) 页面下载 `hugo_extended_xxx_windows-amd64.zip`，解压到 `C:\hugo\`，然后把这个路径添加到系统环境变量 PATH 中

安装完重开终端，确认版本：

```powershell
hugo version
```

## 创建 Hugo 站点

```powershell
hugo new site myblog
cd myblog
git init
```

这会生成一个 Hugo 项目的标准目录结构。

### 安装主题

可用 PaperMod 主题

```powershell
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 配置站点

编辑项目根目录下的 `hugo.toml` 文件：

```toml
baseURL = "https://你的GitHub用户名.github.io/"
languageCode = "zh-cn"
title = "你的博客标题"
theme = "PaperMod"

[params]
  author = "你的名字"
  description = "博客简介"
  ShowReadingTime = true
  ShowPostNavLinks = true
  ShowCodeCopyButtons = true
```

## 写第一篇文章

使用 Hugo 命令创建文章：

```powershell
hugo new posts/my-first-post.md
```

打开 `content/posts/my-first-post.md`，结构如下：

```markdown
---
title: "文章标题"
date: 2026-03-04
draft: false
tags: ["标签1", "标签2"]
---

正文内容，用 Markdown 语法写...
```

**注意：`draft` 字段一定要改成 `false`，否则文章不会被发布。** 这是最常见的"文章写了但网站上看不到"的原因。

### 本地预览

```powershell
hugo server
```

浏览器打开 `http://localhost:1313` 即可预览效果。确认无误后 `Ctrl+C` 停止。

## 部署到 GitHub Pages

### 创建 GitHub 仓库

1. 登录 GitHub，点击右上角 `+` → `New repository`
2. 仓库名必须是 `你的用户名.github.io`（比如用户名是 zhangsan，仓库名就是 `zhangsan.github.io`）
3. 选择 Public
4. **不要**勾选 "Add a README file"
5. 点击 Create repository

### 创建自动部署配置

在项目根目录下创建 GitHub Actions 工作流文件：

```powershell
New-Item -ItemType Directory -Force -Path .github\workflows
```

然后创建 `.github\workflows\deploy.yml` 文件，写入以下内容：

```yaml
name: Deploy Hugo

on:
  push:
    branches:
      - main

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

> **注意：** YAML 文件对缩进非常敏感，必须使用空格而不是 Tab。

### 推送到 GitHub

```powershell
git add .
git commit -m "init blog"
git branch -M main
git remote add origin https://github.com/你的用户名/你的用户名.github.io.git
git push -u origin main
```

推送时会弹出 GitHub 登录窗口进行授权。如果终端要求输入密码，需要使用 Personal Access Token（PAT）而不是 GitHub 密码。生成方式：GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token，勾选 `repo` 权限即可。

### 配置 GitHub Pages

1. 打开你的 GitHub 仓库页面
2. 点击 Settings → 左侧找到 Pages
3. Source 选择 `Deploy from a branch`
4. Branch 选择 `gh-pages`，目录选 `/ (root)`
5. 点击 Save

如果看不到 `gh-pages` 分支，去 Actions 标签页查看构建进度，等绿色勾出现就好了。

等待 1-2 分钟后，访问 `https://你的用户名.github.io` 就能看到你的博客了。

## 日常使用：Git 命令速查

搭建完成后，日常写博客只需要几个 Git 命令。

### 发布新文章

```powershell
hugo new posts/文章名.md        # 创建新文章
# 编辑文章内容，记得把 draft 改为 false
hugo server                     # 本地预览
git add .                       # 暂存所有改动
git commit -m "add: 文章标题"    # 提交
git push                        # 推送，自动触发部署
```

### 修改已有内容

不管是改网站标题、修改文章内容还是调整配置，流程都一样：

```powershell
# 修改文件后
git add .
git commit -m "update: 修改说明"
git push
```

### 查看状态和历史

```powershell
git status                # 查看当前改动了哪些文件
git log --oneline         # 查看提交历史
git diff                  # 查看具体改了什么
```

### commit 信息规范建议

养成写清楚 commit 信息的习惯，方便回溯：

- `add: 新增XXX文章` — 新增文章
- `update: 修改XXX` — 修改已有内容
- `fix: 修复XXX问题` — 修复配置错误
- `config: 调整站点配置` — 修改 hugo.toml 等配置

## 总结

整个流程归纳下来就是：

1. 安装 Git 和 Hugo
2. `hugo new site` 创建站点，安装主题，编辑 `hugo.toml`
3. `hugo new posts/xxx.md` 写文章
4. GitHub 上创建 `用户名.github.io` 仓库
5. 配置 GitHub Actions 自动部署
6. `git add → commit → push` 三连，等一分钟网站自动更新

以后写博客的体验就是：打开编辑器 → 写 Markdown → push → 完事。把时间花在内容上，而不是折腾环境。