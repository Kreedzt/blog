---
author: "Kreedzt"
title: "nodejs npm 在 windows 下的问题"
date: "2023-02-02"
description: "nodejs npm 在 windows 下的问题"
tags: ["web", "javascript", "npm", "nodejs", "windows"]
draft: false
---

# nodejs npm 在 windows 下的问题

## 用户名包含空格时, 执行 npx 报错, 路径不对问题
BUG见: [create-reat-app issue](https://github.com/facebook/create-react-app/issues/9091#issuecomment-678667182)

复现步骤:

```bash
C:\Users\Vaidehi Shah\Desktop\MERN-ShoppingList\client> npx create-react-app .
```
输出:

```
Error: EPERM: operation not permitted, mkdir 'C:\Users\Vaidehi'
```

很明显, 以上用户名被截取了

解决方案:

首先打开 CMD(命令指示符), 进入到用户名上一级目录下, 执行 `dir /x` 命令
```bash
# 通常是如下目录
C:\Users>dir /x
```
```
 驱动器 C 中的卷没有标签。 卷的序列号是 F818-9B1A

 C:\Users 的目录

2020/11/10  19:24    <DIR>                       .
2020/11/10  19:24    <DIR>                       ..
2021/07/24  20:08    <DIR>          KENZHA~1     Ken Zhao
2021/07/08  22:37    <DIR>                       Public
               0 个文件              0 字节
               4 个目录 722,590,040,064 可用字节
```
以上是我的用户名: `Ken Zhao` 左侧就是简写名称: `KENZHA~1`.
复制这个简写名称, 使用 `npm config edit`去编辑一行:
```bash
npm config edit
```
此处的分号表示注释, 删除分号, 改为正确的路径名
修改前:
```
;cache=C:\Users\Ken Zhao\AppData\Roaming\npm-cache
```
修改后:
```
cache=C:\Users\KENZHA~1\AppData\Roaming\npm-cache
```
保存后重新运行 `npx`相关命令, 可以正常工作


