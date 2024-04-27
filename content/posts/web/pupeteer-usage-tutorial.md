---
author: "Kreedzt"
title: "Pupeteer 入门"
date: "2024-04-22"
description: "Pupeteer 使用教程"
tags: ["web", "javascript", "node.js"]
draft: true
---

# Pupeteer 入门

> pupeteer 是 Chrome 的无头浏览器版本, 可以通过 API 操作 Chrome 浏览器操作, 本文通过实现爬虫来阐述简要使用方式

## 启动参数调优

puppeteer 占用资源很大, 需要禁用诸多特性来使其占用降低, 关闭不必要的资源消耗:

> ref: https://pptr.dev/api/puppeteer.puppeteerlaunchoptions

推荐如下参数:

- `--no-sandbox`
- `--disable-setuid-sandbox`
- `--disable-dev-shm-usage`
- `--disable-accelerated-2d-canvas`
- `--no-first-run`
- `--no-zygote`
- `--single-process`
- `--disable-gpu`

```js
  const browser = await puppeteer.launch({
    headless: 'new',
    args: [
      '--no-sandbox',
      '--disable-setuid-sandbox',
      '--disable-dev-shm-usage',
      '--disable-accelerated-2d-canvas',
      '--no-first-run',
      '--no-zygote',
      '--single-process',
      '--disable-gpu'
    ]
  });
```

## 获取数据

> 多线程获取

> 节省资源开销, 在获取所需数据后终止后续请求