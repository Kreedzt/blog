---
author: "Kreedzt"
title: "VSCode 插件 RWR Mod Tool 开发历程(2)"
date: "2024-01-08"
description: "记录 VSCode 插件 RWR Mod Tool 历程"
tags: ["vscode extension", "running with rifles", "mod", "npm", "nodejs", "javascript", "typescript"]
draft: true
---

## 概述

本文主要基于第一篇文档进行的插件迭代更新版本, 版本目标为 0.0.2

## 目标

作为第2版的插件, 当前目标为内容:

- XML 文件引用扫描, 不存在则抛出警告

### 分析

RWR 的文件引用是通过 XML 标签特殊属性值来寻找的文件引用, 如 weapon 寻找 `file`, hud_icon 寻找 `hud_icon` 等. 可通过读取文件 API + 诊断 API 来实现

通过 language 上的 createDiagnosticCollection 创建诊断实例, 标记目标代码位置即可: [API 文档](https://code.visualstudio.com/api/references/vscode-api#languages.createDiagnosticCollection)

### 注册事件