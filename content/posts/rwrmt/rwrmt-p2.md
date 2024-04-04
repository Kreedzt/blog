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

通过注册文件保存 / 文件创建 / 文件删除 / 工作区变更 / 文件重命名回调函数来确保文件扫描触发时机正确性:

[API 参考](https://code.visualstudio.com/api/references/vscode-api#workspace)

```typescript
context.subscriptions.push(
    vscode.workspace.onDidSaveTextDocument((e) => {
        console.log('onDidSaveTextDocument: startXmlCheck');
        startXmlCheck();
    }),
    vscode.workspace.onDidCreateFiles((e) => {
        console.log('onDidCreateFiles: startXmlCheck');
        startXmlCheck();
    }),
    vscode.workspace.onDidDeleteFiles((e) => {
        console.log('onDidDeleteFiles: startXmlCheck');
        startXmlCheck();
    }),
    vscode.workspace.onDidChangeWorkspaceFolders((e) => {
        console.log('onDidChangeWorkspaceFolders: startXmlCheck');
        startXmlCheck();
    }),
    vscode.workspace.onDidRenameFiles((e) => {
        console.log('onDidRenameFiles: startXmlCheck');
        startXmlCheck();
    }),
);
```

### 触发警告

根据 [languages API](https://code.visualstudio.com/api/references/vscode-api#languages) 可知, 创建警告信息需要使用 `createDiagnosticCollection()` 函数,

再根据返回的集合, 直接调用 `set()` 方法即可触发, 其中 key 为 `URI` 类型, 值为 [Diagnostic](https://code.visualstudio.com/api/references/vscode-api#Diagnostic) 类型数组

Diagnostic 中可以指定警告类型 `severity` 来控制标记为 `ERROR` 或 `WARNING`, message 为在 "问题" 窗口中提示的消息内容

示例:
```typescript
const diagnostics = vscode.languages.createDiagnosticCollection();

diagnostics.set(
    uri,
    rangeList.map((r) => {
        return {
            severity: vscode.DiagnosticSeverity.Warning,
            message: `Resource not found: ${r.file}`,
            range: new vscode.Range(
                new vscode.Position(r.line, r.character),
                new vscode.Position(
                    r.line,
                    r.character + r.file.length,
                ),
            ),
        };
    }),
);
```

### 编写扫描代码

本节进行核心代码: XML 文件引用扫描的编写

#### 读取当前所有目标 XML 文件

编写规则, 使用 vscode 读取文件 api, 拿到所有合法文件 uri:

```typescript
const [calls, factions, items, weapons] = await Promise.all([
    vscode.workspace.findFiles('**/calls/*.{xml,call}'),
    vscode.workspace.findFiles('**/factions/*.{models,xml}'),
    vscode.workspace.findFiles('**/items/*.{carry_item,base}'),
    vscode.workspace.findFiles('**/weapons/*.{weapon,xml}'),
]);

const allUri = [...calls, ...factions, ...items, ...weapons];
```

#### 根据 URI 读取 XML 结构化文件内容

```typescript
import { XMLParser } from 'fast-xml-parser';

// 方便对象重用, 降低重复实例化 parser 开销
const xmlParser: {
    parser: XMLParser | null;
    getParser(): XMLParser;
    parse: (content: string) => Record<string, any>;
} = {
    parser: null,
    getParser() {
        if (!this.parser) {
            this.parser = new XMLParser({
                // 关键: 解析属性值
                parseAttributeValue: true,
                ignoreAttributes: false,
                // 允许 boolean 值
                allowBooleanAttributes: true,
                parseTagValue: true,
                commentPropName: 'comment',
                numberParseOptions: {
                    hex: true,
                    leadingZeros: false,
                    eNotation: true,
                },
            });
        }

        return this.parser;
    },
    parse(content: string) {
        return this.getParser().parse(content);
    }
};

const parseXML = (content: string) => {
    return xmlParser.parse(content);
};
```

```typescript
export const scanFile = async (e: vscode.Uri) => {
    const file = await vscode.workspace.fs.readFile(e);
    // 转为字符串处理
    const fileContent = file.toString();
    // 解析结构化 XML
    const xmlStruct = parseXML(fileContent);
    console.log('xmlStruct', xmlStruct);

    // 下一节处理
    const checkRes = await checkRefFileExists(e, fileContent, xmlStruct);
};
```

#### 检测目标属性值的文件是否存在

```typescript
const checkRefFileExists = async (
    e: vscode.Uri,
    fileContent: string,
    struct: any,
): Promise<ICheckRes> => {
    const checkRes: ICheckRes = {
        result: true,
        properties: [],
    };

    const allStructFileRef = extractStructFileRef(struct);

    if (allStructFileRef.length === 0) {
        return checkRes;
    }

    let allFileRefSet = new Set<string>();

    allStructFileRef.forEach((p) => {
        allFileRefSet.add(p.propertyValue);
    });

    const globPattern = `**/{${[...allFileRefSet].join(',')}}`;

    const allFieldsResult = await vscode.workspace.findFiles(globPattern);

    let allAvaiableFileSet = new Set<string>();

    allFieldsResult.forEach((u) => {
        const sp = u.path.split('/');
        const fileName = sp[sp.length - 1];
        allAvaiableFileSet.add(fileName);
    });

    allStructFileRef.forEach((p) => {
        const result = allAvaiableFileSet.has(p.propertyValue);

        if (!result) {
            checkRes.result = false;
        }
        checkRes.properties.push({
            name: p.propertyName,
            file: p.propertyValue,
            result: allAvaiableFileSet.has(p.propertyValue),
        });
    });

    // mark error
    const rangeList: Array<{
        line: number;
        character: number;
        file: string;
    }> = [];
    checkRes.properties
        .filter((p) => !p.result)
        .forEach((property) => {
            const pos = getAllPosition(fileContent, property.file);

            pos.forEach((p) => {
                rangeList.push({
                    line: p.line,
                    character: p.character,
                    file: property.file,
                });
            });
        });

    if (!checkRes.result) {
        FileResResolver.self().addMissingFileWarn(e, rangeList);
    }

    return checkRes;
};
```