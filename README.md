# 几时 - 个人技术博客

基于 [Hexo](https://hexo.io/) 框架 + [NexT](https://theme-next.org/) 主题搭建的个人博客，部署在 GitHub Pages 上。

在线访问：https://hiram711.github.io/blog

## 环境要求

- [Node.js](https://nodejs.org/)（建议 v12+）
- [Git](https://git-scm.com/)

## 快速开始

### 1. 安装依赖

```bash
npm install
```

> 如果安装缓慢，可先切换为国内镜像：
> ```bash
> npm config set registry https://registry.npmmirror.com
> ```

### 2. 本地预览

```bash
hexo server
```

启动后访问 http://localhost:4000/blog 即可预览博客。

### 3. 生成静态文件

```bash
hexo generate
```

生成的文件位于 `public/` 目录。

### 4. 部署到 GitHub Pages

```bash
hexo deploy
```

也可以一步完成生成和部署：

```bash
hexo g -d
```

部署目标为 `gh-pages` 分支。

## 写作

### 新建文章

```bash
hexo new "文章标题"
```

文件会创建在 `source/_posts/` 目录下，使用 Markdown 格式编写。

文章头部的 Front Matter 格式：

```yaml
---
title: 文章标题
author: 几时西风
date: 2026-04-16 10:00:00
tags:
  - 标签1
  - 标签2
categories:
  - 分类名
---
```

### 新建草稿

```bash
hexo new draft "草稿标题"
```

草稿保存在 `source/_drafts/` 目录，不会被发布。

### 发布草稿

```bash
hexo publish "草稿标题"
```

将草稿移动到 `source/_posts/` 并正式发布。

### 图片引用

将图片放入 `source/images/` 目录，在文章中引用：

```markdown
![图片描述](/blog/images/your-image.png)
```

## 项目结构

```
blog/
├── _config.yml          # Hexo 主配置文件
├── package.json         # 项目依赖
├── scaffolds/           # 文章模板
│   ├── post.md          #   发布文章模板
│   ├── draft.md         #   草稿模板
│   └── page.md          #   页面模板
├── source/              # 内容源文件
│   ├── _posts/          #   已发布文章
│   ├── _drafts/         #   草稿
│   ├── _discarded/      #   已废弃文章
│   ├── images/          #   图片资源
│   ├── about/           #   关于页面
│   ├── categories/      #   分类页面
│   ├── tags/            #   标签页面
│   └── 404/             #   404 页面
└── themes/next/         # NexT 主题
    └── _config.yml      #   主题配置文件
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `hexo server` | 启动本地服务器 |
| `hexo generate` | 生成静态文件 |
| `hexo deploy` | 部署到远程 |
| `hexo clean` | 清除缓存和已生成的静态文件 |
| `hexo new "标题"` | 新建文章 |
| `hexo new draft "标题"` | 新建草稿 |
| `hexo publish "标题"` | 发布草稿 |

## 配置说明

- 站点配置：编辑根目录下的 `_config.yml`
- 主题配置：编辑 `themes/next/_config.yml`
- 管理后台：安装了 `hexo-admin` 插件，启动服务器后访问 http://localhost:4000/blog/admin/

## 分支说明

- `hexo`：源文件分支（默认开发分支）
- `gh-pages`：构建产物分支（GitHub Pages 部署）
