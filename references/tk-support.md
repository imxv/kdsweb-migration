# TK（Tachikoma）端支持

本文档说明迁移到一码多投 2.0 后，新增 TK 端支持所需的文件和配置。

> **TK 支持是可选的**。在迁移流程的 Step 1c 中会询问用户是否需要 TK 支持。以下所有内容仅在用户选择添加 TK 支持时适用。

## 目录

1. [src/index.tk.ts](#1-srcindextkts--tk-入口文件) — TK 入口
2. [demos/index.entry.ts](#2-demosindexentryts--tk-demo-入口) — TK Demo 入口
3. [tsconfig.tk.json](#3-tsconfigtkjson--tk-专用-typescript-配置)
4. [package.json "tk" 字段](#4-packagejson-tk-字段)
5. [kds.config.js TK 配置](#5-kdsconfigjs-tk-配置项)
6. [TK devDependencies](#6-tk-所需的-devdependencies)
7. [TK 组件源码改造指南](#7-tk-端组件源码改造指南) — Solid.js 重写判断标准和速查表

---

## 1. `src/index.tk.ts` — TK 入口文件

TK 端的包入口，通过 package.json 的 `"tk"` 字段指向此文件。

```ts
export { default } from './component';
```

**说明**：
- 与 `src/index.ts`（RN 入口）内容相同
- 文件名必须以 `.tk.ts` 结尾，TK 打包器通过 `mainFields: ["tk"]` 解析到此文件
- 如果组件有 TK 端特殊逻辑，可创建 `.tk.tsx` 后缀的同名文件实现平台差异

---

## 2. `demos/index.entry.ts` — TK Demo 入口

TK 端的 demo 启动文件，使用 Tachikoma API 注册视图。

```ts
import { create } from '@kds/native-js/tk';
import Demo1 from './demo1';

// 如果有多个demo, 在这里添加多份，比如使用 pageName=demo1 参数区分不同demo
export const demoMap = {
  demo1: Demo1,
};

Object.keys(demoMap).forEach((viewKey) => {
  // @ts-ignore
  Tachikoma.registerView(viewKey, () => {
    return create(demoMap[viewKey]);
  });
});
```

**说明**：
- `create` 来自 `@kds/native-js/tk`，将 KDS 组件转为 TK 可渲染的视图
- `Tachikoma` 是 TK 运行时全局对象，`@ts-ignore` 是必需的（该全局变量无 TS 声明）
- `registerView(key, factory)` 注册视图工厂函数，key 对应 URL 参数中的页面标识
- Demo 文件（如 `demo1.tsx`）在 KRN/Web/TK 三端共享，无需单独编写

---

## 3. `tsconfig.tk.json` — TK 专用 TypeScript 配置

```json
{
  "compilerOptions": {
    "baseUrl": "./",
    "outDir": "./dist/types",
    "noImplicitAny": true,
    "module": "ESNext",
    "target": "ESNext",
    "strict": false,
    "sourceMap": false,
    "allowJs": false,
    "skipLibCheck": true,
    "moduleResolution": "node",
    "allowSyntheticDefaultImports": true,
    "jsx": "preserve",
    "jsxImportSource": "@kds/native-js",
    "types": ["@kds/tachikoma-lib"],
    "lib": [
      "dom",
      "dom.iterable",
      "esnext"
    ],
    "esModuleInterop": true,
    "downlevelIteration": true,
    "paths": {
      "@/*": [
        "src/*"
      ],
      "@es/<package-name>": [
        "src/index.ts"
      ]
    },
    "declaration": true,
    "strictNullChecks": true
  },
  "include": [
    "src/**/*",
    "demos/**/*",
    "entry/**/*"
  ],
  "exclude": [
    "node_modules",
    "build",
    "**/*.test.ts"
  ]
}
```

**与主 `tsconfig.json` 的关键区别**：

| 配置项 | 主 tsconfig | tsconfig.tk.json | 原因 |
|--------|------------|-----------------|------|
| `jsx` | `"react"` | `"preserve"` | TK 使用自定义 JSX 转换 |
| `jsxImportSource` | 无 | `"@kds/native-js"` | TK 的 JSX 运行时来源 |
| `types` | 无 | `["@kds/tachikoma-lib"]` | 提供 Tachikoma 全局类型 |
| `module` | `"es6"` | `"ESNext"` | TK 需要 ESNext 模块 |
| `target` | `"es5"` | `"ESNext"` | TK 运行时支持现代语法 |
| `noImplicitAny` | 无 | `true` | TK 构建要求 |
| `allowJs` | `true` | `false` | TK 仅处理 TS 文件 |

---

## 4. package.json `"tk"` 字段

在 package.json 顶层添加：

```json
{
  "tk": "./src/index.tk.ts"
}
```

**说明**：
- 此字段告诉 TK 打包器使用哪个文件作为包入口
- 类似于 `"react-native"` 字段指定 RN 入口的机制
- 放置位置建议在 `"react-native"` 字段之后

---

## 5. kds.config.js TK 配置项

在 `kds.config.js` 中添加 `tk` 配置块：

```js
const path = require('path');
module.exports = {
  krn: { /* ... 原有 KRN 配置 ... */ },
  tk: {
    "$schema": "https://unpkg.corp.kuaishou.com/@kds/native-shared/dist/schema.json",
    "version": "1.0.0",
    "versionCode": 1,
    "scheme": "kwai://",
    "inlineRequire": true,
    "grammarCheckIngore": ["flat"],
    "ssgOption": {},
    "mainFields": ["tk", "module", "main"],
    "extensions": [
      ".tk.tsx", ".tk.ts", ".tk.jsx", ".tk.js",
      ".tsx", ".ts", ".jsx", ".js",
      "..."
    ],
    "src": "demos",
    "tsConfigPath": path.join(process.cwd(), 'tsconfig.tk.json'),
    "includeBabelPackages": [
      "@kds/web",
      "@kds/web-*",
      "@es/kprom-*",
    ],
    "plugins": [require.resolve('@es/babel-plugin-kope-tk')],
  }
}
```

**关键配置说明**：
- `mainFields`: 解析包入口的优先级，`"tk"` 最优先
- `extensions`: 文件扩展名解析顺序，`.tk.*` 后缀优先
- `src`: demo 文件目录
- `tsConfigPath`: 指向 TK 专用 tsconfig
- `includeBabelPackages`: 需要 babel 转换的包列表（node_modules 中默认不转换）

---

## 6. TK 所需的 devDependencies

```json
{
  "@kds/native-js": "^0.2.2",
  "@es/babel-plugin-kope-tk": "^1.0.1-alpha.4",
  "@kds/babel-plugin-jsx-dom-expressions": "^0.2.12",
  "@kds/tachikoma-extra": "^1.2.8",
  "@kds/tachikoma-lib": "^0.9.6"
}
```

详见 [dependency-changes.md](./dependency-changes.md)。

---

## 7. TK 端组件源码改造指南

### 7a. 判断标准：是否需要手写 `.tk.tsx`

并非所有组件都需要手写 TK 版本。判断标准：

| 组件类型 | 是否需要手写 `.tk.tsx` | 说明 |
|---------|----------------------|------|
| **简单组件**（纯展示、少量 props、无状态或简单状态） | **不需要** | 依赖 `@es/babel-plugin-kope-tk` 编译转换即可 |
| **复杂组件**（多个 state/effect 交互、ref 操作、复杂生命周期） | **需要手写** | 编译转换无法正确处理复杂的 React hooks 交互，必须用 Solid.js 原语重写 |

**复杂度判断信号**：
- 组件内有 3 个以上 `useState`
- 存在 `useEffect` 依赖其他 state 的链式更新
- 使用 `useRef` + `useImperativeHandle` 暴露实例方法
- 使用 `React.forwardRef` + `React.memo` 组合
- 有复杂的条件渲染逻辑依赖多个 state

### 7b. Solid.js 原语速查表

手写 `.tk.tsx` 时，需要将 React hooks 转换为 Solid.js 响应式原语：

| React (xxx.tsx) | Solid/TK (xxx.tk.tsx) | 说明 |
|---|---|---|
| `useState(init)` | `createSignal(init)` | 返回 `[getter, setter]`，getter 是**函数**，取值需 `getter()` |
| `useMemo(() => x, [deps])` | `createMemo(() => x)` | 无需手动声明依赖，自动追踪 |
| `useEffect(() => { ... }, [deps])` | `createEffect(() => { ... })` | 自动追踪依赖，无需手动声明 |
| `useCallback(fn, [deps])` | `fn`（普通函数即可） | Solid 不需要 useCallback，函数不会因重新渲染而重建 |
| `useRef(init)` | `let ref = init` | 普通变量即可，组件不会重新执行 |
| `React.forwardRef((props, ref) => ...)` | 普通函数组件 `(props) => ...` | Solid 不需要 forwardRef |
| `React.memo(Component)` | 不需要 | Solid 的响应式系统天然细粒度更新 |
| `state` 直接取值 | `state()` 函数调用取值 | **最常见的错误来源**：signal 取值必须调用函数 |

### 7c. `.tk.tsx` 文件注意事项

1. **JSX pragma**：TK 端 `.tk.tsx` 文件顶部需要声明 JSX 运行时来源：
   ```tsx
   /** @jsxRuntime automatic */
   /** @jsxImportSource @kds/native-js */
   ```

2. **Props 不可解构**：Solid.js 的 props 是响应式代理对象，**解构会丢失响应性**：
   ```tsx
   // ❌ 错误：解构会丢失响应性
   const MyComponent = ({ title, onPress }) => { ... }

   // ✅ 正确：通过 props.xxx 访问
   const MyComponent = (props) => {
     return <View onPress={props.onPress}><Text>{props.title}</Text></View>
   }
   ```

3. **入口文件**：`src/index.tk.ts` 对于简单组件可以与 `src/index.ts` 内容相同。但对于复杂组件，如果组件文件有对应的 `.tk.tsx` 版本，TK 打包器会通过 `extensions` 配置自动解析到 `.tk.tsx` 文件，因此 `src/index.tk.ts` 的 export 路径通常无需修改。
