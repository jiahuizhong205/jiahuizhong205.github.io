---
title: 我的博客搭建记录
date: 2026-09-01 12:00:00 +0800
tags: [博客, GitHub Pages, Jekyll]
categories: [折腾]
---

想有一个自己的博客，记录学习和踩坑。对比了几种方案后，我选了 **Jekyll + Chirpy 主题**，托管在 GitHub Pages 上——不用买服务器、不用备案，写完 Markdown 推上去就能发布。这第一篇blog就用来记录记录这次从零到上线的过程，也把踩过的坑记下来。


## 搭建步骤

### 1. 用模板创建仓库

打开 [chirpy-starter](https://github.com/cotes2020/chirpy-starter)，点 **Use this template** 创建新仓库，仓库名必须是 `<用户名>.github.io`（比如 `jiahuizhong205.github.io`）。

### 2. 开启 GitHub Actions 构建

进入仓库 **Settings → Pages**，Source 选 **GitHub Actions**。这一步让仓库自带的 workflow 接管构建和发布。

> 注意：Chirpy 用的是 GitHub Actions 构建，**不是** GitHub Pages 内置的旧版 Jekyll——因为它的功能需要自定义插件。

### 3. 修改配置

编辑根目录的 `_config.yml`，改这几个关键项（更多的项可以后续自行探索优化）：

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

之后就能在编辑器里改文件、写文章（_posts下新建.md文件即可），push 后 Actions 自动构建发布。

## 踩坑：文章不显示

搭好后我新建了一篇 `Hello-World.md`，push 上去却发现首页不显示。排查后发现，Jekyll 识别文章有两个硬性要求：

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
这第一篇blog有很多技术细节是由AI帮忙补充完善的，查看代码仓库里的本markdown文件可以更直观的看到真实的文章结构要求。

后续还会在本文中更新更多配置、美化细节（因为我自己也在摸索呜呜）————2026.9.1

