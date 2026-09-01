---
title: 我的博客搭建记录
date: 2026-09-01 12:00:00 +0800
tags: [博客, GitHub Pages, Jekyll]
categories: [折腾]
---

想有一个自己的博客，记录学习和踩坑。对比了几种方案后，我选了 **Jekyll + Chirpy 主题**，托管在 GitHub Pages 上——不用买服务器、不用备案，写完 Markdown 推上去就能发布。这篇记录这次从零到上线的过程，也把踩过的坑记下来。

## 为什么是 Jekyll + Chirpy

主流的静态博客方案不少：

| 方案 | 语言 | 特点 |
| --- | --- | --- |
| Hugo | Go | 构建最快，单二进制 |
| Hexo | Node.js | 中文社区活跃，主题多 |
| Jekyll | Ruby | GitHub Pages 原生支持 |
| Astro | TypeScript | 现代前端栈 |

我选 Jekyll 的原因很简单：**GitHub Pages 内置了它**，push 上去就能自动构建发布，省去「本地 build → 部署产物」这一步。主题选了 Chirpy，因为它专门为技术博客设计，深色模式、搜索、目录、评论这些刚需开箱即用，而且作者是中文开发者，文档好查。

## 搭建步骤

### 1. 用模板创建仓库

打开 [chirpy-starter](https://github.com/cotes2020/chirpy-starter)，点 **Use this template** 创建新仓库，仓库名必须是 `<用户名>.github.io`（比如 `jiahuizhong205.github.io`）。

### 2. 开启 GitHub Actions 构建

进入仓库 **Settings → Pages**，Source 选 **GitHub Actions**。这一步让仓库自带的 workflow 接管构建和发布。

> 注意：Chirpy 用的是 GitHub Actions 构建，**不是** GitHub Pages 内置的旧版 Jekyll——因为它的功能需要自定义插件。

### 3. 修改配置

编辑根目录的 `_config.yml`，改这几个关键项：

```yaml
lang: zh-CN              # 中文界面
timezone: Asia/Shanghai  # 时区
title: 你的博客名
url: 'https://<用户名>.github.io'
```

### 4. 拉到本地用编辑器写

```bash
git clone https://github.com/<用户名>/<用户名>.github.io.git
```

之后就能在编辑器里改文件、写文章，push 后 Actions 自动构建发布。

## 踩坑：文章不显示

搭好后我新建了一篇 `Hello-World.md`，push 上去却发现首页根本不显示。排查后发现，Jekyll 识别文章有两个硬性要求：

1. **文件名必须有日期前缀**：`Hello-World.md` ❌ → `2026-09-01-Hello-World.md` ✅
2. **顶部必须有 front matter**：

```yaml
---
title: 文章标题
date: 2026-09-01 12:00:00 +0800
tags: [博客]
---
```

少了任何一样，Jekyll 都会把文件当成普通文件跳过。

## 设置头像

头像在 `_config.yml` 里配置。把图片放到 `assets/img/` 目录下，然后填：

```yaml
avatar: /assets/img/avatar.png
```

## 小结

从零到上线，全程没装任何本地环境，主要时间花在等 Actions 构建上。以后写文章就是「新建带日期前缀的 md 文件 → push」两件事。
