---
author: "Kreedzt"
title: "vitest 项目集成 codecov"
date: "2023-07-09"
description: "适用于Github actions: 为 vitest 项目集成 codecov, 使其在 pull_request 与 push 时集成"
tags: ["web", "javascript", "codecov", "github actions", "github"]
draft: false
---

## 项目配置

### vitest 配置

不需要特殊配置, 增加 coverage 配置即可:

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    // 注意: 新版 vitest 默认 watch: true, 需要修改为 false, 否则命令会始终不停止
    watch: false,
    // ...
    coverage: {
      include: ['src'],
      all: true,
      provider: 'istanbul',
      reporter: ['text', 'json', 'html'],
    }
  },
});
```

### npm 命令

为使其 bash 脚本运行报告, 需要添加 npm script:

```json
{  
  "scripts": {
    "coverage": "vitest --coverage"
  }
}
```


### 配置 github actions

> 参考: https://docs.codecov.com/docs/github-2-getting-a-codecov-account-and-uploading-coverage#github-actions

配置 github actions 为配置 ci 的触发规则, codecov 配置需要额外配置

实例参考:

```yml
# 在任意分支的 push 与 pull_request 执行
on: [push, pull_request]

# Github actions 权限
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  test:
    runs-on: ubuntu-latest
    name: Test Coverage
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      - name: Setup Nodejs
        uses: actions/setup-node@v2
        with:
          node-version: '18'
      # 安装依赖项
      - name: Install
        run: npm i -g pnpm && pnpm i
      # 修改为你的测试步骤
      - name: test coverage
        run: pnpm run coverage
      - name: Upload coverage reports to Codecov
        uses: codecov/codecov-action@v3
        # 对于私有项目, 会需要 token, 公开项目无需此配置
        with:
          token: ${{ secrets.CODECOV_TOKEN }}

```

### codecov 配置

在项目根目录编写 `codecov.yml` 即可进行 codecov 配置, 即为运行 codecov 所需的配置:

如果需要在发起 pull_request 时自动运行测试覆盖率报告, 可用如下官方参考配置:

> 参考: https://docs.codecov.com/docs/pull-request-comments

```yml
comment:                  # this is a top-level key
  layout: " diff, flags, files"
  behavior: default
  require_changes: false  # if true: only post the comment if coverage changes
  require_base: false        # [true :: must have a base report to post]
  require_head: true       # [true :: must have a head report to post]
```

第一次运行效果: (因为初次没有上一次的参考, 所以目标比较值为 `?`)
![初次效果](../images/codecov_example1.png)

之后的运行效果:
![之后效果](../images/codecov_example2.png)

## 项目 Badges 设置

> 参考文档: https://docs.codecov.com/docs/status-badges

通常在 codecov 网站上项目设置可找到嵌入代码:

```markdown
[![codecov](https://codecov.io/gh/Kreedzt/bbr-server-stats/branch/master/graph/badge.svg?token=E5289JZXpe)](https://codecov.io/gh/Kreedzt/bbr-server-stats)
```

实际效果:
![badge效果](../images/codecov_example3.png)
