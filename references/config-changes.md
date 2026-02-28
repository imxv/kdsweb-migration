# 配置文件变更指南

> 本文档中 1.0 的定义见 [SKILL.md](../SKILL.md)。

## 目录

1. [demos/index.tsx 模板](#1-demosindextsx-krn-demo-入口模板) — KRN 入口
2. [demos/index.web.tsx 模板](#2-demosindexwebtsx-web-demo-入口模板) — Web 入口
3. [babel.config.js](#3-babelconfigjs-变更) — 完整模板
4. [kds.config.js](#4-kdsconfigjs-变更) — KRN + TK 配置
5. [tsconfig.json](#5-tsconfigjson-变更) — 排除 browser 目录
6. [.gitignore](#6-gitignore-变更)
7. [package.json scripts](#7-tk-packagejson-scripts-变更仅在选择添加-tk-支持时需要) — [TK]
8. [config/default.js externals](#8-configdefaultjs--externalsmoduleskeys-更新)
9. [config/default.js bundless](#9-tk-configdefaultjs--liboptionsbundless-tk-配置仅在选择添加-tk-支持时需要) — [TK]
10. [.dumirc.ts](#10-dumircts--headscripts-cdn-url-更新)

---

## 1. `demos/index.tsx`（KRN Demo 入口）模板

```tsx
import { AppRegistry } from 'react-native';

import React, { useMemo } from 'react';
import { setGlobalConfig, getUrlParam } from '@kds/web-api';

import Demo1 from './demo1';

// 如果有多个demo, 在这里添加多份，比如使用 pageName=demo1 参数区分不同demo
const Components = {
  demo1: Demo1,
};

const App = (props) => {
  useMemo(() => {
    // @ts-ignore
    setGlobalConfig({ pageProps: props }); // 必须
  }, [props]);

  const pageName = getUrlParam('pageName') as string;

  const Comp = Components[pageName] || Demo1;

  return <Comp />;
};

AppRegistry.registerComponent('Kwaishop', () => App);
```

**注意**：
- `setGlobalConfig` 的 `IGlobalConfig` 类型要求传入 `logger` 字段，但 demo 中不需要，所以必须加 `// @ts-ignore`
- `getUrlParam()` 返回类型为 `unknown`，需要 `as string` 断言

---

## 2. `demos/index.web.tsx`（Web Demo 入口）模板

与 `demos/index.tsx` 类似，但使用 `react-dom/client` 的 `createRoot`：

```tsx
import React, { useMemo } from "react";
import { getUrlParam, setGlobalConfig } from "@kds/web-api";
import { createRoot } from "react-dom/client";
import Demo1 from "./demo1";

const Components = {
  'demo1': Demo1,
};

const App = (props) => {
  useMemo(() => {
    // @ts-ignore
    setGlobalConfig({ pageProps: props }); // 必须
  }, [props]);

  const pageName = getUrlParam("pageName") as string;

  // @ts-ignore
  const Comp = Components[pageName] || Demo1;

  return <Comp />;
};

// ... 其余 Web 初始化代码
```

---

## 3. `babel.config.js` 变更

确保使用 `@kds/web-shared`：

```js
// 2.0 目标状态
const { KOPE_PLATFORM } = require('@kds/web-shared');
```

### 完整 babel.config.js 模板（供参考）

```js
const { KOPE_PLATFORM } = require('@kds/web-shared');

module.exports = function (api) {
  api.cache(true);

  const presets = [];
  const plugins = [];

  if (KOPE_PLATFORM === 'krn') {
    // KRN 端配置
    presets.push('module:metro-react-native-babel-preset');
  } else if (KOPE_PLATFORM === 'tk') {
    // TK 端配置
    presets.push(['@babel/preset-env', { targets: { node: 'current' } }]);
    presets.push('@babel/preset-typescript');
    plugins.push(['@es/babel-plugin-kope-tk']);
    plugins.push(['@kds/babel-plugin-jsx-dom-expressions']);
  } else {
    // Web 端 / 默认配置
    presets.push(['@babel/preset-env', { targets: { node: 'current' } }]);
    presets.push('@babel/preset-typescript');
    presets.push(['@babel/preset-react', { runtime: 'automatic' }]);
  }

  return { presets, plugins };
};
```

> **注意**：以上模板仅供参考，实际项目的 babel.config.js 可能有额外的插件和配置。迁移时主要关注：
> 1. 确保顶部使用 `require('@kds/web-shared')` 获取 `KOPE_PLATFORM`
> 2. TK 分支中确保包含 `@es/babel-plugin-kope-tk` 和 `@kds/babel-plugin-jsx-dom-expressions` 插件

---

## 4. `kds.config.js` 变更

### 4a. KRN 配置块

将原 `krn.config.json` 的内容移入 `kds.config.js` 的 `krn` 字段。

**1.0**：使用独立的 `krn.config.json` 文件
**2.0**：合并到 `kds.config.js` 中

```js
const path = require('path');
module.exports = {
  krn: {
    "entry": { "Kwaishop": "./demos/index.tsx" },
    "scheme": "kwai://",
    "projectName": "Kwaishop",
    "installCmd": "yarn",
    "framework": "React",
    "language": "TypeScript",
    "schemeParams": {
      "Kwaishop": "bgColor=%23FF000000&title=KProM&themeStyle=1"
    }
  },
  tk: { /* ... 见 §5 ... */ }
}
```

### 4b. `[TK]` TK 配置块（仅在选择添加 TK 支持时需要）

详见 [tk-support.md](./tk-support.md) §5。

---

## 5. `tsconfig.json` 变更

### 5a. 排除 `src/browser/` 目录

`src/browser/` 目录可能引用已删除的旧的 1.0 依赖，会导致 TS 编译错误。在 `exclude` 中添加：

```json
{
  "exclude": [
    "node_modules",
    "build",
    "src/browser/**",
    "**/*.test.ts"
  ]
}
```

### 5b. 保持 `skipLibCheck: true`

确保 `skipLibCheck` 为 `true`，避免 node_modules 中的类型冲突影响构建。

---

## 6. `.gitignore` 变更

如果迁移前使用了 `krn.config.json`，现在已合并到 `kds.config.js`，可以删除或添加到 `.gitignore`。

---

## 7. `[TK]` `package.json` scripts 变更（仅在选择添加 TK 支持时需要）

新增 TK 相关 script：

```json
{
  "scripts": {
    "start:tk": "KOPE_PLATFORM=tk tk start"
  }
}
```

---

## 8. `config/default.js` — externalsModulesKeys 更新

更新 externals 列表，添加新包：

**新增**：
```js
'@kds/web',
'@kds/web-api',
```

完整示例：

```js
const externalsModulesKeys = [
  ...Object.keys(pkg.peerDependencies),
  'react-dom',
  'react-native-linear-gradient',
  'react-native-svg',
  'react-native',
  'react',
  '@es/request',
  '@kds/web',
  '@kds/web-api',
  '@kid-ui/krn',
  '@krn/cli',
  '@yoda/bridge',
  '@kds/bridge-lite',
  '@ks/weblogger',
  '@ks/yoda-js-sdk',
  '@ks/yoda-kuaishou-plugin',
  '@es/kprom-common-image',
];
```

---

## 9. `[TK]` `config/default.js` — libOptions.bundless TK 配置（仅在选择添加 TK 支持时需要）

为 TK 端构建添加 bundless 配置：

```js
libOptions: {
  bundless: {
    ignores: ['**/browser/**'],
    disableJSXToJsExtension: true,
    builtinBabelPresetsHandler(presets) {
      return presets.filter(preset => {
        return !preset[0]?.includes('@babel/preset-react');
      });
    }
  },
},
```

**说明**：
- `disableJSXToJsExtension: true`：TK 使用自定义 JSX 转换，不要将 `.jsx`/`.tsx` 改为 `.js`
- `builtinBabelPresetsHandler`：过滤掉 `@babel/preset-react`，因为 TK 使用 `@kds/native-js` 的 JSX 运行时而非 React
- `ignores: ['**/browser/**']`：排除 browser 目录下的文件

---

## 10. `.dumirc.ts` — headScripts CDN URL 更新

如果 headScripts 中引用了 KSnack embed.js 的 CDN URL，确保使用最新版本：

```ts
headScripts: [
  {
    src: 'https://cdnfile.corp.kuaishou.com/kc/files/a/kwaishop-web-doc/KProMKSnack/embed.a6d1619bd7514890.js',
    async: true,
  },
],
```

**注意**：URL 中的 hash 值可能会更新，请以内部文档中的最新值为准。
