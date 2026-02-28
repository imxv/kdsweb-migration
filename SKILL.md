---
name: kdsweb-migration
description: >
  将 KDS/KProM 组件从一码多投 1.0（RTX）迁移到 2.0（@kds/web 系列）。
  1.0 项目基于 react-native + @es/rtx-api，组件从 react-native 导入，API 从 @es/rtx-api 导入。
  触发条件：用户要求迁移 KDS 组件、kprom 组件、一码多投组件到 2.0 或 @kds/web 时使用。
  包括：(1) 包名和 import 路径替换 (2) Touchable 组件替换为 View+onPress
  (3) 类型迁移（Props 派生） (4) API 签名变化和 Bridge 调用迁移
  (5) 可选 TK（Tachikoma）端支持 (6) 配置文件和依赖管理
---

# KDS-Web 迁移 Skill：一码多投 1.0 → 2.0

迁移路径：`react-native` / `@es/rtx-api` → `@kds/web` / `@kds/web-api`

---

## 前置条件

- 本地开发环境已安装 `yarn`
- 有 `jia` CLI 工具（`@es/jia-config`）

---

## Step 1: 版本检测与项目分析（必须最先执行）

**在进行任何修改之前，必须先判断项目当前的一码多投版本。**

### 1a. 检测步骤

依次执行以下检查：

1. 读取 `package.json`，检查 `dependencies`、`devDependencies`、`peerDependencies` 三个区域中的包名
2. 扫描 `src/` 和 `demos/` 下所有 `.ts`/`.tsx`/`.js`/`.jsx` 文件的 import 语句
3. **检查 1.0 标志**：peerDependencies 中是否同时包含 `react-native`（或 `react-native-web`）和 `@es/rtx-api`
4. **检查源码中的 `react-native` 组件导入**：是否存在 `import { View, Text, StyleSheet, Platform, ... } from 'react-native'`（注意区分组件导入与 `AppRegistry` 等 API 导入）

### 1b. 版本判定规则

根据检测结果，按以下规则判定版本：

| 判定结果 | 条件 |
|---------|------|
| **1.0（需要迁移）** | 满足以下**全部条件**：①`peerDependencies` 中包含 `react-native`（或 `react-native-web`）等 1.0 标志性依赖；② 源码中存在从 `react-native` 导入 UI 组件的语句（如 `View`、`Text`、`StyleSheet`、`Platform` 等）；③ 不存在任何 `@kds/web*` 包或 import |
| **2.0（无需迁移）** | `peerDependencies` 中包含 `@kds/web`（或 `@kds/web-api`），**并且**源码中的 UI 组件从 `@kds/web` 导入（而非从 `react-native`） |
| **混合状态（需要继续迁移）** | 同时存在 1.0 和 2.0 的引用特征（部分迁移），例如同时存在 `react-native` 组件导入与 `@kds/web` 组件导入 |

> **1.0 判定补充说明**：
> 一码多投 1.0（RTX）项目直接使用 `react-native` 导入 UI 组件（`View`、`Text` 等），配合 `@es/rtx-api` 提供 `rem`、`setPageProps`、`getUrlParam` 等工具函数。这类项目的典型 peerDependencies 特征为包含 `react-native`、`react-native-web`、`@es/rtx-api` 等，但不含 `@kds/web*`。
>
> **注意区分**：`import { AppRegistry } from 'react-native'` 属于 KRN 端运行时 API，**不是** 1.0 的判定依据。判定依据是 **UI 组件**（`View`、`Text`、`Image`、`ScrollView`、`StyleSheet`、`Platform` 等）是否从 `react-native` 导入。

### 1c. 版本判定后的行为

- **如果判定为 2.0（无需迁移）**：**立即停止，不做任何文件修改**。向用户报告检测结果：
  > 检测到该项目已经是一码多投 2.0 版本（使用 `@kds/web` 系列包），无需进行迁移。

  然后列出检测到的证据（如使用的 `@kds/web*` 包列表），结束流程。

- **如果判定为 1.0 或混合状态**：向用户报告检测结果，列出需要迁移的具体项。然后**询问用户是否需要同时添加 TK（Tachikoma）端支持**，并根据用户的选择决定后续步骤中是否执行 TK 相关操作（标记为 `[TK]` 的步骤）。之后继续执行 Step 2 及后续步骤。

### 1d. 完整分析（仅在需要迁移时执行）

如果判定需要迁移，还需补充以下分析：

1. 记录当前版本号和所有 1.0 依赖的完整列表（`react-native`、`@es/rtx-api` 等）
2. 读取 `src/` 目录结构，识别所有源码文件
3. 统计所有 `.ts`/`.tsx` 文件中需要替换的 import（文件名 + 行号）：统计从 `react-native` 导入 UI 组件的语句（`View`、`Text`、`StyleSheet`、`Platform`、`Image`、`ScrollView` 等），以及从 `@es/rtx-api` 导入 API 的语句（`rem`、`setPageProps`、`getUrlParam`、`env` 等）
4. 检查是否存在 `react-native` 类型导入（`ViewStyle`、`TextStyle` 等）
5. 检查是否存在 `src/browser/` 目录（可能引用旧包）

---

## Step 2: 升级 TypeScript（最高优先级前置步骤）

**必须在任何代码改动之前完成**。

`@kds/web` 使用了 TypeScript 4.5+ 的 inline type import 语法（`import type { X } from 'y'`），`skipLibCheck: true` **无法跳过语法解析错误**。低版本 TypeScript 会直接报语法错误导致构建失败。

```json
{
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

详见 [references/dependency-changes.md](./references/dependency-changes.md) §2a。

---

## Step 3: 更新依赖

按以下顺序操作：

1. **新增 `@kds/web*` 依赖到 devDependencies**：添加 `@kds/web`、`@kds/web-api` 等项目源码中实际使用到的包
2. **删除 `@types/react-native`**：会与 `@kds/web` 类型冲突
3. **`[TK]` 添加 TK 依赖**：`@kds/native-js`、`@es/babel-plugin-kope-tk`、`@kds/babel-plugin-jsx-dom-expressions`、`@kds/tachikoma-extra`、`@kds/tachikoma-lib`
4. **更新 peerDependencies**：新增 `@kds/web: *`、`@kds/web-api: *`；移除 `react-native-web`（2.0 不再需要作为 peerDependency）
5. **更新版本号**：`(X+1).0.0`（如 `2.0.1` → `3.0.0`，主版本递增因为 peer dependency 变了）
6. **`[TK]` 添加 `"tk"` 字段**：`"tk": "./src/index.tk.ts"`
7. **`[TK]` 添加 scripts**：`"start:tk": "KOPE_PLATFORM=tk tk start"`

> **Monorepo 首次迁移注意**：如果这是 monorepo 中第一个迁移到 2.0 的组件，需要先在**monorepo 根目录**安装公共依赖：
> ```bash
> yarn add -W @kds/web @kds/web-api
> ```
> 后续迁移的其他组件不需要重复此步骤。

详见 [references/dependency-changes.md](./references/dependency-changes.md)。

---

## Step 4: 修改源码

### 4a. import 路径全局替换

将 `react-native` 的 UI 组件/工具类 import 改为 `@kds/web`，Touchable 系列替换为 `View` + `onPress`。注意 `AppRegistry` 和第三方 RN 扩展包保持不变。

**注意**：以下 `react-native` 导入**保持不变**，不要替换：
- `import { AppRegistry } from 'react-native'` — KRN 端运行时 API，在 demo 入口文件中使用
- 第三方 react-native 扩展包（如 `react-native-linear-gradient`、`react-native-svg`）— 保持原有导入

详见 [references/source-code-changes.md](./references/source-code-changes.md) §1-§6。

### 4b. 不支持属性处理（nativeProps 透传）

**⚠️ 核心原则：绝对不能直接删除** react-native 中存在但 @kds/web 类型定义中不存在的组件属性。

`@kds/web` 组件提供了 `nativeProps` 机制来透传平台特有的属性。处理流程：
1. 如果有统一属性替代（如 `showsVerticalScrollIndicator` → `showScrollIndicator`），使用替代属性
2. 无统一替代时，移入 `nativeProps.rn`（如 `stickyHeaderIndices`、`selectionColor`）
3. 仅当确认所有端都不需要时才可删除

详见 [references/source-code-changes.md](./references/source-code-changes.md) §2。

### 4c. 类型迁移

**推荐方式**：从 `@kds/web` 的 Props 派生样式类型：

```ts
import type { ViewProps, TextProps } from '@kds/web';

type MyProps = {
  containerStyle?: ViewProps['style'];  // 替代 ViewStyle
  textStyle?: TextProps['style'];       // 替代 TextStyle
};
```

**不要**保留从 `react-native` 导入的 `ViewStyle`/`TextStyle` 类型。

详见 [references/source-code-changes.md](./references/source-code-changes.md) §7。

### 4d. babel.config.js

确保 `babel.config.js` 使用 `require('@kds/web-shared')`。

---

## Step 5: 修改 Demo 文件

### 5a. `demos/demo1.tsx`

替换 import 路径即可。

### 5b. `demos/index.tsx`（KRN 入口）

替换 import 路径，并注意两个类型问题：
- `setGlobalConfig({ pageProps: props })` 需要 `// @ts-ignore`
- `getUrlParam('pageName')` 需要 `as string`

### 5c. `demos/index.web.tsx`（Web 入口）

同 5b。

详见 [references/config-changes.md](./references/config-changes.md) §1-§2。

---

## Step 6: 修改配置文件

### 6a. `kds.config.js`

如果项目原来使用 `krn.config.json`，将其内容合并到 `kds.config.js` 的 `krn` 字段。**`[TK]`** 如果用户选择添加 TK 支持，还需添加 `tk` 配置块。

详见 [references/config-changes.md](./references/config-changes.md) §4。

### 6b. `config/default.js`

1. **更新 externalsModulesKeys**：添加 `@kds/web`、`@kds/web-api`，移除旧包
2. **`[TK]` 添加 bundless TK 配置**：`disableJSXToJsExtension` + `builtinBabelPresetsHandler`

详见 [references/config-changes.md](./references/config-changes.md) §8-§9。

### 6c. `tsconfig.json`

在 `exclude` 中添加 `"src/browser/**"`（如果该目录存在且引用旧包）。

详见 [references/config-changes.md](./references/config-changes.md) §5。

---

## Step 7: `[TK]` 创建 TK 支持文件

> **仅在用户选择添加 TK 支持时执行此步骤。** 如果用户不需要 TK 支持，跳过此步骤。

创建以下 4 个文件：

1. **`src/index.tk.ts`** — TK 入口
2. **`demos/index.entry.ts`** — TK Demo 入口
3. **`tsconfig.tk.json`** — TK TypeScript 配置

> **复杂组件注意**：如果组件包含复杂状态逻辑（多个 useState/useEffect 交互、ref 操作等），TK 的编译转换可能无法正确处理，需要**手写 `.tk.tsx` 版本**，用 Solid.js 响应式原语（`createSignal`、`createEffect`、`createMemo`）重写。简单组件（纯展示或简单状态）依赖编译转换即可。详见 [references/tk-support.md](./references/tk-support.md) §7。

详见 [references/tk-support.md](./references/tk-support.md)。

---

## Step 8: 处理遗留文件

### 8a. `src/browser/` 目录

此目录可能引用已删除的旧的 1.0 依赖，会导致 TS 编译报错。

**方案 A（推荐）**：在 `tsconfig.json` 的 `exclude` 中添加 `"src/browser/**"`
**方案 B**：确认不需要后整个目录删除

### 8b. `krn.config.json`

如果已合并到 `kds.config.js`，删除此文件。

---

## Step 9: 安装依赖

```bash
yarn install
```

---

## Step 10: 验证

### 10a. Web 构建验证（最重要）

```bash
jia build
```

**这是最关键的验证步骤**。`jia dev` 不会暴露所有类型错误，只有 `jia build` 会进行完整的 TS 类型检查和打包。

> **注意**：首次运行 `jia build` 可能报 `@es/jia-config` 相关错误，这是暂时性问题，**直接重试一次即可**。

### 10b. Web 开发服务验证

```bash
jia dev
```

确认页面正常渲染，无运行时报错。

### 10c. `[TK]` TK 构建验证

> 仅在添加了 TK 支持时执行。

```bash
KOPE_PLATFORM=tk tk start
```

### 10d. KRN 验证

```bash
krn start
```

---

## 常见问题排查

| # | 问题 | 原因 | 解决方案 |
|---|------|------|---------|
| 1 | `Parsing error: '>' expected` 或 TS 语法错误 | TypeScript 版本过低 | 升级 `typescript` 到 `^5.0.0`（[Step 2](#step-2-升级-typescript最高优先级前置步骤)） |
| 2 | `Type 'ViewStyle' is not assignable to...` | `@types/react-native` 类型冲突 | 删除 `@types/react-native`，使用 `ViewProps['style']`（[Step 3](#step-3-更新依赖)） |
| 3 | `Property 'alignSelf' does not exist on type...` | KDS TextStyle 不含 flex 属性 | 给样式对象加 `as any`（[source-code-changes.md §7b](./references/source-code-changes.md)） |
| 4 | `Argument of type '{ pageProps: any; }' is not assignable to 'IGlobalConfig'` | `setGlobalConfig` 要求 `logger` 字段 | 添加 `// @ts-ignore`（[Step 5b](#5b-demosindextskrn-入口)） |
| 5 | `Type 'unknown' is not assignable to type 'string'` | `getUrlParam()` 返回 `unknown` | 添加 `as string`（[Step 5b](#5b-demosindextskrn-入口)） |
| 6 | `Cannot find name 'Tachikoma'` | TK 全局变量无类型声明 | `[TK]` 添加 `// @ts-ignore`（[tk-support.md §2](./references/tk-support.md)） |
| 7 | `src/browser/` 中的编译错误 | 引用了已删除的旧包 | 在 tsconfig.json exclude 中添加 `"src/browser/**"`（[Step 8a](#8a-srcbrowser-目录)） |
| 8 | `jia build` 首次失败报 `@es/jia-config` 错误 | 暂时性缓存问题 | 直接重试 `jia build` 即可 |
| 9 | TK demo 白屏或运行时错误 | 缺少 TK 支持文件或配置 | `[TK]` 检查 `src/index.tk.ts`、`demos/index.entry.ts`、`tsconfig.tk.json` 是否正确创建（[Step 7](#step-7-tk-创建-tk-支持文件)） |

---

## 参考文件

- [references/dependency-changes.md](./references/dependency-changes.md) — 依赖变更详情
- [references/source-code-changes.md](./references/source-code-changes.md) — 源码变更详情
- [references/config-changes.md](./references/config-changes.md) — 配置文件变更详情
- [references/tk-support.md](./references/tk-support.md) — TK 端支持文件模板

---

## KProM 组件迁移快速检查清单

迁移 `@es/kprom-*` 组件时，按以下清单逐项确认：

1. **源码 import**：`react-native` UI 组件 → `@kds/web`
2. **Touchable 组件**：`TouchableOpacity`/`TouchableWithoutFeedback` → `View` + `onPress`（[source-code-changes.md §6b](./references/source-code-changes.md)）
3. **不支持属性处理**：**绝对不能直接删除**。有统一替代的用替代属性（如 `showsVerticalScrollIndicator` → `showScrollIndicator`），无统一替代的移入 `nativeProps.rn`（如 `stickyHeaderIndices`、`selectionColor`）（[source-code-changes.md §2](./references/source-code-changes.md)）
4. **类型**：不要无脑替换为 `any`，用 `Props['style']` 和 `Parameters<>` 精确推导（[source-code-changes.md §7](./references/source-code-changes.md)）
5. **API**：`@es/rtx-api` → `@kds/web-api`；`setPageProps()` → `setGlobalConfig({ pageProps })`；**删除 `setRemBaseLine`**
6. **Bridge**：`bridge.canIUse/invoke` → `invoke('ns.method', params)`（[source-code-changes.md §9](./references/source-code-changes.md)）
7. **日志**：组件库中的 `debugLogger` 直接移除或改为 `console.warn`，不要依赖宿主工程日志路径（[source-code-changes.md §10](./references/source-code-changes.md)）
8. **package.json**：版本号主版本迭代、peerDeps 更新、删除 `@types/react-native`、添加 `typescript: ^5.0.0`
9. **`[TK]` TK 支持**（可选）：简单组件靠编译转换即可；复杂组件需手写 `.tk.tsx`（[tk-support.md §7](./references/tk-support.md)）
10. **Monorepo 首次迁移**：根目录 `yarn add -W @kds/web @kds/web-api`

---

## 1.0 与 2.0 混用说明

迁移后的 2.0 组件**可以在 1.0 的工程中继续使用**。前提条件：
- 宿主工程安装 `@kds/web >= 1.1.5`
- 宿主工程的 babel 配置中包含 adapter 插件

这意味着组件可以先行迁移到 2.0，不需要等待所有消费方工程同步迁移。
