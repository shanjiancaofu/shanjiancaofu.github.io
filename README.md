# 山间草夫

费福朝的个人技术博客，主要记录 C++、Linux、RT-Thread、嵌入式、AI、算法与项目开发学习过程。

访问地址：<https://shanjiancaofu.github.io>

## 本地运行

项目基于 [Jekyll](https://jekyllrb.com/) 和 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 主题。

### Docker 运行

在 WSL2 项目目录运行：

```shell
docker compose -f docker/docker-compose.yml up --build
```

浏览器访问 <http://127.0.0.1:4000>。

停止服务：

```shell
docker compose -f docker/docker-compose.yml down
```

### 本机 Ruby 运行

如果不用 Docker，也可以安装 Ruby 和 Bundler 后运行：

```shell
bundle install
bundle exec jekyll serve
```

浏览器访问 <http://127.0.0.1:4000>。

## 写作

文章放在 `_posts` 目录，文件名格式为：

```text
YYYY-MM-DD-title.md
```

文章头部模板：

```yaml
---
title: 文章标题
date: 2026-06-14 20:00:00 +0800
categories: [文章分类]
tags: [标签一, 标签二]
---
```

目前使用的分类：

- `RT-Thread`：内核机制、线程管理、同步通信、内存与中断
- `Linux`：常用命令、文件系统、Makefile 和开发工具链
- `嵌入式`：STM32、ADC/DAC、回调函数与底层开发笔记
- `算法`：算法基础、控制算法和题目总结
- `AI`：本地部署、模型压缩和大模型学习记录
- `C++`：语言基础、STL 和工程实践
- `随笔`：博客建设、学习记录和阶段复盘

`categories` 保持简单，一个文章使用一个主要分类；`tags` 用于标记具体知识点。

提交到 `main` 或 `master` 分支后，GitHub Actions 会自动构建并部署博客。
