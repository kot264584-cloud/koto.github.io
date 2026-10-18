# Koto Portfolio

Koto 的个人作品集站点，使用 Hugo Extended、PaperMod 和 GitHub Pages 构建。

线上地址：

```text
https://kot264584-cloud.github.io/koto.github.io/
```

## 本地预览

```powershell
hugo server -D
```

打开 `http://localhost:1313/`。

## 添加内容

美术作品：

```powershell
hugo new content art/work-name/index.md --kind art
```

技术 Demo：

```powershell
hugo new content technical/demo-name/index.md --kind technical
```

图形笔记：

```powershell
hugo new content notes/note-name.md --kind notes
```

新建后编辑 Markdown 文件，将 `draft` 改为 `false`。图片可以直接放在同目录的页面包中，并在 front matter 中设置 `cover.image`。

## 目录

```text
content/art/          美术作品
content/technical/    技术 Demo
content/notes/        图形学笔记
content/about.md      个人介绍
static/cv/            简历 PDF
static/images/        全站图片
static/videos/        短视频素材
assets/css/extended/  自定义样式
```

## 发布

推送到 `main` 后，GitHub Actions 会自动构建并部署到 GitHub Pages。

首次使用时，在 GitHub 仓库的 `Settings -> Pages` 中将 `Source` 设为 `GitHub Actions`。

如果以后把仓库重命名为 `kot264584-cloud.github.io`，需要同时修改 `hugo.toml` 中的 `baseURL`。
