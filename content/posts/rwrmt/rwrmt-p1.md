---
author: "Kreedzt"
title: "VSCode 插件 RWR Mod Tool 开发历程(1)"
date: "2024-01-08"
description: "记录 VSCode 插件 RWR Mod Tool 历程"
tags: ["vscode extension", "running with rifles", "mod", "npm", "nodejs", "javascript", "typescript"]
---

## 立项缘由

Running with rifles(以下简称 `RWR`) 的 mod 开发目前存在诸多问题:
- 人形绑骨麻烦
- 多人开发时 XML 格式不统一
- 缺少文件引用检查
- 各 xml 文件定义属性作用未知
- 注册方式容易遗漏
- ...

VSCode 作为大多数 Mod 开发者使用的工具, 起草 VSCode 插件目的是逐步解决如上问题.

已知的 VSCode 插件可解决内容:
- 多人开发时 XML 格式不统一: 通过统一格式化处理
- 各 xml 文件定义属性作用未知: 通过定义模板命令 / 代码片段处理
- 缺少文件引用检查: 通过扫描文件引用 key 来查找工作空间所有文件名引用, 未找到抛出警告

本系列文章逐步尽可能解决所有已知问题

> 本文主要描述第一版本的开发内容

## 项目启动

新注册 VSCode 插件项目, 按照 VSCode extension 官方教程即可:

https://code.visualstudio.com/api/get-started/your-first-extension

该教程会引导注册一个 "命令", 命令在 VSCode 中用以 `Ctrl(or command)-Shift-P` 启动的命令


## 目标

作为第一版的插件, 目标仅定以下内容:
- 注册命令
  + 创建武器模板
  + 创建护甲模板
- 插件打包及发布

## 注册激活条件

我们不需要在任意文件结构的目录都激活插件, 仅需要在 mod 目录即可.

可使用 glob 表达式来限定工作区的文件, 若匹配才激活

参考:

https://code.visualstudio.com/api/references/activation-events#workspaceContains

在 package.json 中写入以下内容, 限定插件激活条件:

```json
{
  "activationEvents": [
    "workspaceContains:**/calls/all_calls.xml",
    "workspaceContains:**/factions/all_factions.xml",
    "workspaceContains:**/items/all_carry_items.xml",
    "workspaceContains:**/weapons/all_weapons.xml"
  ]
}
```

## 注册命令

> https://code.visualstudio.com/api/extension-guides/command#creating-new-commands

注册命令仅需按照模板, 调用 `vscode.commands.registerCommand` 即可:
```typescript
context.subscriptions.push(
    vscode.commands.registerCommand(
        'vscode-rwr-mod-tool.createWeapon',
        async () => {
            // ...
        },
    ),
);
```

如果想要允许在命令面板中使用, 需要在 package.json 中注册交互命令:
> 参考: https://code.visualstudio.com/api/extension-guides/command#creating-a-user-facing-command

```json
{
  "contributes": {
    "commands": [
      {
        "command": "vscode-rwr-mod-tool.createArmor",
        "title": "RWR Mod Tool: Create Armor"
      },
      {
        "command": "vscode-rwr-mod-tool.createWeapon",
        "title": "RWR Mod Tool: Create Weapon"
      },
    ],
  }
}
```

## 完善创建武器逻辑

RWR 核心注册武器流程为:
1. 编写 `*.weapon` 文件
2. 注册到 `all_weapons.xml` 中

创建武器核心逻辑:
1. 弹出输入框, 用户输入新武器名称
2. 弹出选择框, 用户选择注册的目标文件
   - 如果 all_weapons.xml 中列举的为文件引用(属性值如: `file="*.xml"`, 且目标文件包含 `<weapons>` 标签), 那么需要引导用户注册到这一层文件中
   
### 获取用户输入的武器名称

> https://code.visualstudio.com/api/references/vscode-api#window

通过 window api 可弹出交互式输入框:

```typescript
const inputVal = await vscode.window.showInputBox({
    title: 'Weapon name',
    placeHolder: 'Enter your weapon name, do not input .xml suffix',
    validateInput: (val) => {
        if (val.trim().length === 0) {
            return 'Please input weapon name';
        }

        return '';
    },
});
```

### 解析 all_weapons.xml, 引导用户选择注册目标文件

vscode 支持 glob 表达式来获取文件:
https://code.visualstudio.com/api/references/vscode-api#workspace

```typescript
import * as vscode from 'vscode';

export const getAllWeaponsUri = async (): Promise<undefined | vscode.Uri> => {
    const uris = await vscode.workspace.findFiles('**/weapons/all_weapons.xml');
    let targetUri: vscode.Uri | undefined = undefined;

    uris.forEach((u) => {
        if (u.path.endsWith('/weapons/all_weapons.xml')) {
            targetUri = u;
        }
    });

    console.log('in getAllWeaponsFolderUri:');
    console.log(uris);

    return targetUri;
};
```

通过 findFiles 获取的结果为 `Uri[]` 类型, 用以后续的文件内容读取:

```typescript
const filePath = await getAllWeaponsUri();
if (!filePath) {
    return [];
}

const fileContent = (
    await vscode.workspace.fs.readFile(filePath)
).toString();
```

### 获取注册目标列表

使用 fast-xml-parser 库解析 XML

**注意**: 需要允许布尔值, 且解析属性值:

```typescript
import { XMLParser } from 'fast-xml-parser';

export const parseXML = (content: string) => {
    const parser = new XMLParser({
        parseAttributeValue: true,
        ignoreAttributes: false,
        allowBooleanAttributes: true,
        parseTagValue: true,
        commentPropName: 'comment',
        numberParseOptions: {
            hex: true,
            leadingZeros: false,
            eNotation: true,
        },
    });

    return parser.parse(content);
};
```

解析的属性值默认携带 `@_` 前缀, 通过 `file` 属性来获取目标的武器注册文件列表:

```typescript
const xml = parseXML(fileContent);

// Parse all_weapons.xml data
const weaponNames: string[] = [];

xml.weapons.weapon.forEach((weapon) => {
    weaponNames.push(weapon['@_file']);
});
```

### 弹出选择器引导用户选择, 并更新注册目标文件

https://code.visualstudio.com/api/references/vscode-api#window

调用 `showQuickPick()` 函数来引导用户选择:

```typescript
// Select all_weapons.xml data
const select = await vscode.window.showQuickPick(weaponNames);
```


解析目标 xml 文件, 合并用户输入的新项:
```typescript
const uris = await vscode.workspace.findFiles(`**/${select}`);

const groupFileName = uris[0];
if (!groupFileName) {
    return false;
}

// 读取目标文件
const groupFileContent = (
    await vscode.workspace.fs.readFile(groupFileName)
).toString();

const groupFileXml = parseXML(groupFileContent);

// 插入新项
const newWeaponList = [
    ...groupFileXml.weapons.weapon,
    {
        // inputWeaponName 为用户输入的新项
        '@_file': `${inputWeaponName}.weapon`,
    } as IWeaponRegisterXML,
];
groupFileXml.weapons.weapon = newWeaponList;

// 构建新 XML 结构
const newGroupFileXmlContent = buildXML(groupFileXml);

// 写入文件
await vscode.workspace.fs.writeFile(
    groupFileName,
    Buffer.from(newGroupFileXmlContent),
);
```

### 新建模板文件

新建模板文件流程:

1. 获取目标目录
2. 生成模板代码
3. 写入文件
4. 打开文件, 方便用户编辑

```typescript
// 获取 all_weapons.xml 对应的 uri
const targetUri = await getAllWeaponsUri();
if (!targetUri) {
    return;
}

// 写入目标为同级目录
const writePath = vscode.Uri.joinPath(
    vscode.Uri.joinPath(targetUri, '../'),
    `${weaponName}.weapon`,
);

// 替换模板
const xmlContent = TemplateService.getCls().getXMLContent({ weaponName });

if (!xmlContent) {
    return;
}

// 写入文件
await vscode.workspace.fs.writeFile(writePath, Buffer.from(xmlContent));

// 打开文件
const textDocument = await vscode.workspace.openTextDocument(writePath);
await vscode.window.showTextDocument(textDocument, {
    preview: true,

});
```

## 配置发布流程

### 打包

参考: https://code.visualstudio.com/api/working-with-extensions/publishing-extension

当使用 pnpm 初始化时, 直接执行 vsce package 会出现[如下问题](https://github.com/microsoft/vscode-vsce/issues/421)

```txt
npm ERR! missing ....
```

虽然 vsce 支持使用 pnpm 初始化作为包管理器, 但是发布流程不支持直接操作.

解决方案见: https://github.com/microsoft/vscode-vsce/issues/421#issuecomment-1038911725

### 注册发布账号

1. 首先需要[创建项目组织](https://learn.microsoft.com/zh-cn/azure/devops/organizations/accounts/create-organization?view=azure-devops), 组织旗下可以存在多个发布者
2. 按照 [文档获取 Access Token](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#get-a-personal-access-token)
3. [创建发布者](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#create-a-publisher)
4. 使用 vsce 凭借 Token 登录

至此, 配置完毕, `vsce login` 后无需二次登录操作

### 发布扩展程序

按照[文档](https://code.visualstudio.com/api/working-with-extensions/publishing-extension#publish-an-extension), 存在 2 种发布方式, 为方便发布, 之后使用 `vsce` 直接发布

### 补充

1. 版本根据 `package.json` 中的 `version` 指定即可
2. 插件会使用 `README.md` 来作为预览文档
3. `README.md` 文档插入**仅支持 https 格式外链或 base64 格式**, 无法使用相对路径图片等资源(见 [issue](https://github.com/microsoft/vscode-vsce/issues/390))
4. `engines` 字段默认取当前 vscode 版本, 测试时需要注意更新 vscode 版本

vscode 插件 package.json 使用字段参考: https://code.visualstudio.com/api/references/extension-manifest