# 源码变更指南

> 本文档中 1.0 的定义见 [SKILL.md](../SKILL.md)。

## 目录

1. [Import 路径替换](#1-import-路径替换) — 包名映射表（react-native → @kds/web）
2. [不支持属性处理（nativeProps 机制）](#2-不支持属性处理nativeprops-机制) — **绝对不能直接删除属性**
3. [API 变化](#3-api-import-变化) — API 签名差异、setRemBaseLine 删除
4. [react-native 特殊处理](#6-react-native-import-处理) — Touchable 组件替换
5. [类型迁移](#7-类型迁移最关键改动) — Props 派生、速查表、TS 错误
6. [Demo 文件修改](#8-demo-文件修改) — KRN/Web 入口、env.isWeb 拆分
7. [Bridge 调用迁移](#9-bridge-调用迁移1-0-b-关键改动) — canIUse/invoke 签名变化
8. [组件库日志处理](#10-组件库日志处理debuglogger) — debugLogger 策略

---

## 1. import 路径替换

### 1.0 → 2.0（react-native → @kds/web）

将源码中从 `react-native` 导入的 **UI 组件和工具** 替换为 `@kds/web`：

| 旧 import | 新 import | 说明 |
|-----------|-----------|------|
| `import { View, Text, Image, ScrollView, ... } from 'react-native'` | `import { View, Text, Image, ScrollView, ... } from '@kds/web'` | UI 组件 |
| `import { StyleSheet, Platform, Dimensions, PixelRatio } from 'react-native'` | `import { StyleSheet, Platform, Dimensions, PixelRatio } from '@kds/web'` | 工具类 |
| `import { NativeEventEmitter, NativeModules } from 'react-native'` | `import { NativeEventEmitter, NativeModules } from '@kds/web'` | RN 原生模块桥接 |
| `import { Keyboard, UIManager, findNodeHandle, Clipboard } from 'react-native'` | `import { Keyboard, UIManager, findNodeHandle, Clipboard } from '@kds/web'` | 其他工具 API |
| `import type { ViewStyle, TextStyle } from 'react-native'` | `import type { ViewProps, TextProps } from '@kds/web'` | 类型（见 §7） |

> **`@kds/web` 从 `react-native` 重新导出的完整列表**（均需替换为 `@kds/web`）：
> - UI 组件：`View`、`Text`、`Image`、`ImageBackground`、`TextInput`、`ScrollView`、`Modal`
> - 工具类：`StyleSheet`、`Platform`、`Dimensions`、`PixelRatio`
> - 原生模块：`NativeEventEmitter`、`NativeModules`、`Keyboard`、`UIManager`、`findNodeHandle`、`Clipboard`
> - 流程控制：`For`、`Show`、`Switch`、`Match`（@kds/web 新增，非 react-native 原有）
> - 动画：`Animated`、`Easing`
> - 其他：`DeviceEventEmitter`、`Linking`、`AppRegistry`、`I18nManager`、`BackHandler`、`StatusBar`

**不需要替换的 `react-native` 导入**：
- `import { AppRegistry } from 'react-native'` — KRN 端运行时 API（注意：@kds/web 也导出了 AppRegistry，但 demo 入口中通常保留从 react-native 导入）
- 第三方 react-native 扩展包（如 `react-native-linear-gradient`、`react-native-svg`）

### @kds/web 中**不存在**的 react-native 导出（不能替换，需要特殊处理）

以下组件/API **在 `@kds/web` 中不存在**（包括最新版 1.1.10），不能简单替换 import 路径：

| react-native 导出 | 处理方式 | 说明 |
|---|---|---|
| `FlatList` | **保留 `import { FlatList } from 'react-native'` 不变** | @kds/web 没有提供 FlatList 组件。如果使用了 FlatList，保留从 react-native 导入即可，KRN 端正常工作。如需跨端列表方案，可考虑使用 `@es/kprom-common-list-view` 或用 ScrollView + 手动逻辑替代 |
| `SectionList` | **保留 `import { SectionList } from 'react-native'` 不变** | 同 FlatList，@kds/web 未提供 |
| `VirtualizedList` | **保留 `import { VirtualizedList } from 'react-native'` 不变** | 同 FlatList |

### @kds/web-api 中**不存在**的旧 API（需要删除，不能迁移）

以下旧 API **在 `@kds/web-api` 中不存在**，不能迁移到 `@kds/web-api`，**必须直接删除**：

| 旧 import | 处理方式 | 说明 |
|---|---|---|
| `import { debugLogger } from '@es/rtx-api'` | **直接删除 import 和所有调用代码** | `debugLogger` 在 2.0 中不是独立导出的 API。2.0 的日志通过 `setGlobalConfig({ logger })` 传入 logger 实例，由框架内部使用。组件库中的 debugLogger 调用应改为 `console.warn` 或直接移除（详见 §10） |
| `import { setRemBaseLine } from '@es/rtx-api'` | **直接删除 import 和所有调用代码** | 2.0 不再需要手动设置 rem 基准值 |

> **判断原则**：除了 `AppRegistry` 之外，所有从 `react-native` 导入的内容都应替换为 `@kds/web`。包括 UI 组件、工具类、原生模块桥接（`NativeEventEmitter`、`NativeModules` 等）。不要因为某个导出"看起来像 RN 运行时 API"就保留在 `react-native`——只有 `AppRegistry` 是例外。

---

## 2. 不支持属性处理（nativeProps 机制）

### 核心原则：**绝对不能直接删除** react-native 中存在但 @kds/web 类型定义中不存在的属性

`@kds/web` 的组件类型定义是**跨端统一接口**，只包含所有端（Web/KRN/TK）都通用的属性。但这**不代表**特定端不支持的属性就应该删除——删除会改变 KRN 端的运行时行为，可能导致视觉或交互回退。

### 2a. nativeProps 透传机制

`@kds/web` 的所有核心组件（View、Text、ScrollView、TextInput、Image 等）都提供了 `nativeProps` 属性，用于向特定端透传平台特有的 props：

```tsx
<ScrollView
  // @kds/web 统一接口中的属性
  horizontal={false}
  showScrollIndicator={false}
  // 通过 nativeProps 透传 RN 特有属性
  nativeProps={{
    rn: { stickyHeaderIndices: [0] },
    web: { /* Web 特有属性 */ },
    tk: { /* TK 特有属性 */ },
  }}
>
```

**结构说明**：
- `nativeProps.rn` — 传递给 RN 端底层 react-native 组件的原生属性
- `nativeProps.web` — 传递给 Web 端底层 HTML 元素的属性
- `nativeProps.tk` — 传递给 TK 端底层组件的属性

### 2b. 属性迁移决策流程

遇到 react-native 中存在但 `@kds/web` 统一接口中不存在的属性时，按以下顺序判断：

1. **检查是否有统一属性替代**（见下方速查表）→ 使用替代属性
2. **无统一替代 → 移入 `nativeProps.rn`**（保持 KRN 端行为不变）
3. **仅当确认该属性在所有端都无意义时** → 才可以删除（需注释说明原因）

### 2c. 常见属性迁移速查表

#### ScrollView

| react-native 旧属性 | @kds/web 处理方式 | 说明 |
|---|---|---|
| `showsVerticalScrollIndicator={false}` | `showScrollIndicator={false}` | **有统一替代**。@kds/web 用 `showScrollIndicator` 统一控制，RN 端内部根据 `horizontal` 自动映射为 `showsVerticalScrollIndicator`/`showsHorizontalScrollIndicator` |
| `showsHorizontalScrollIndicator={false}` | `showScrollIndicator={false}` | 同上 |
| `stickyHeaderIndices={[0]}` | `nativeProps={{ rn: { stickyHeaderIndices: [0] } }}` | **移入 nativeProps.rn**。@kds/web 统一接口不含此属性，但 KRN 端仍支持 |
| `keyboardShouldPersistTaps="handled"` | `nativeProps={{ rn: { keyboardShouldPersistTaps: "handled" } }}` | 移入 nativeProps.rn |
| `nestedScrollEnabled={true}` | `nativeProps={{ rn: { nestedScrollEnabled: true } }}` | 移入 nativeProps.rn |
| `pagingEnabled={true}` | `nativeProps={{ rn: { pagingEnabled: true } }}` | 移入 nativeProps.rn |
| `removeClippedSubviews={true}` | `nativeProps={{ rn: { removeClippedSubviews: true } }}` | 移入 nativeProps.rn |

#### TextInput

| react-native 旧属性 | @kds/web 处理方式 | 说明 |
|---|---|---|
| `selectionColor="#ff0000"` | `nativeProps={{ rn: { selectionColor: "#ff0000" } }}` | **移入 nativeProps.rn**。@kds/web 统一接口不含此属性 |
| `textAlignVertical="top"` | `nativeProps={{ rn: { textAlignVertical: "top" } }}` | 移入 nativeProps.rn |
| `underlineColorAndroid="transparent"` | `nativeProps={{ rn: { underlineColorAndroid: "transparent" } }}` | 移入 nativeProps.rn |
| `autoCorrect={false}` | `nativeProps={{ rn: { autoCorrect: false } }}` | 移入 nativeProps.rn |
| `autoCapitalize="none"` | `nativeProps={{ rn: { autoCapitalize: "none" } }}` | 移入 nativeProps.rn |

#### View

| react-native 旧属性 | @kds/web 处理方式 | 说明 |
|---|---|---|
| `collapsable={false}` | `nativeProps={{ rn: { collapsable: false } }}` | 移入 nativeProps.rn |
| `accessible={true}` | `nativeProps={{ rn: { accessible: true } }}` | 移入 nativeProps.rn |
| `accessibilityLabel="xxx"` | `nativeProps={{ rn: { accessibilityLabel: "xxx" } }}` | 移入 nativeProps.rn |
| `hitSlop={{ top: 10, ... }}` | `nativeProps={{ rn: { hitSlop: { top: 10, ... } } }}` | 移入 nativeProps.rn |

### 2d. nativeProps 合并写法

当同一个组件有多个需要透传的 RN 属性时，合并到一个 `nativeProps` 对象中：

```tsx
// ❌ 错误：直接删除不支持的属性
<ScrollView horizontal={false}>
  {/* stickyHeaderIndices 被删了，KRN 端行为改变 */}
</ScrollView>

// ✅ 正确：移入 nativeProps.rn
<ScrollView
  horizontal={false}
  showScrollIndicator={false}
  nativeProps={{
    rn: {
      stickyHeaderIndices: [0],
      keyboardShouldPersistTaps: 'handled',
    },
  }}
>
```

> **注意**：如果组件原来已有 `nativeProps` 属性，需要合并而不是覆盖。

---

## 3. API import 变化

```ts
// 1.0（使用 @es/rtx-api 的早期项目）
import { rem, setPageProps, getUrlParam, env } from '@es/rtx-api';
// 2.0
import { rem, setGlobalConfig, getUrlParam } from '@kds/web-api';
```

> **注意事项**：
> - `@es/rtx-api` 的 `setPageProps(props)` 对应 2.0 的 `setGlobalConfig({ pageProps: props })`，API 签名不同
> - `@es/rtx-api` 的 `env.isWeb` 在 2.0 中不再需要（Web/KRN 入口文件已分离）
> - 如果组件源码（`src/`）中仅使用 `rem` 函数，可保留 `@es/rtx-api` 的 `rem` 导入不变，仅在 demo 入口文件中替换为 `@kds/web-api`
> - **`setRemBaseLine` 删除**：1.0-B 项目中可能存在 `import { setRemBaseLine } from '@es/rtx-api'; setRemBaseLine(414);`，这些代码在 2.0 中已不需要，**应直接删除整行**（包括 import 和调用）

---

## 6. `react-native` import 处理

### 需要替换的 `react-native` 导入（仅 1.0-B 模式）

在 1.0-B 项目中，以下从 `react-native` 导入的 **UI 组件和工具类** 需要替换为 `@kds/web`：

- `View`、`Text`、`Image`、`ScrollView`、`Modal`、`ImageBackground`、`TextInput` 等 UI 组件
- `StyleSheet`、`Platform`、`Dimensions`、`PixelRatio` 等工具类
- `NativeEventEmitter`、`NativeModules`、`Keyboard`、`UIManager`、`findNodeHandle`、`Clipboard` 等原生模块桥接和工具 API
- `TouchableOpacity`、`TouchableHighlight`、`TouchableWithoutFeedback`（**注意：这些组件在 `@kds/web` 中不存在，需要替换为 `View` + `onPress`，见下方 §6b**）

> **⚠️ 注意：`FlatList`、`SectionList` 在 @kds/web 中不存在**，保留从 `react-native` 导入不变。详见 §1 中的「不存在的 react-native 导出」表。

```ts
// 1.0-B 旧代码
import { View, Text } from 'react-native';
import { StyleSheet, Platform } from 'react-native';
import { NativeEventEmitter, NativeModules } from 'react-native';
// 2.0 新代码
import { View, Text } from '@kds/web';
import { StyleSheet, Platform } from '@kds/web';
import { NativeEventEmitter, NativeModules } from '@kds/web';
```

### 保持不变的 `react-native` 导入（所有模式通用）

以下 `react-native` 导入在任何模式下都**保持不变**：

```ts
import { AppRegistry } from 'react-native';
```

`AppRegistry` 是 KRN 端的运行时 API，仅在 demo 入口文件（`demos/index.tsx`）中使用，不需要替换。

### 第三方 react-native 扩展包

第三方 react-native 扩展包的导入保持不变：

```ts
import LinearGradient from 'react-native-linear-gradient';
import Svg from 'react-native-svg';
```

这些包有独立的 Web alias（如 `react-native-linear-gradient` → `react-native-web-linear-gradient`），在构建配置中处理。

### 6b. Touchable 系列组件替换（1.0-B 关键改动）

`TouchableOpacity`、`TouchableHighlight`、`TouchableWithoutFeedback` 等组件**在 `@kds/web` 中不存在**，不能简单替换 import 路径。必须将它们替换为 `View` + `onPress` 的写法：

```tsx
// ❌ 错误：@kds/web 中没有 TouchableOpacity
import { TouchableOpacity } from '@kds/web';

// ✅ 正确：TouchableOpacity → View + onPress
// 旧代码
<TouchableOpacity onPress={handlePress}>...</TouchableOpacity>
// 新代码
<View onPress={handlePress}>...</View>

// ✅ 正确：带透明度反馈的 TouchableOpacity → View + onPress + activeOpacity
// 旧代码
<TouchableOpacity onPress={handlePress}>...</TouchableOpacity>
// 新代码
<View onPress={handlePress} activeOpacity={0.5}>...</View>

// ✅ 正确：TouchableWithoutFeedback → View + onPress + activeOpacity={0}（无视觉反馈）
// 旧代码
<TouchableWithoutFeedback onPress={handlePress}>...</TouchableWithoutFeedback>
// 新代码
<View onPress={handlePress} activeOpacity={0}>...</View>

// ✅ 正确：TouchableHighlight → View + onPress
// 旧代码
<TouchableHighlight onPress={handlePress}>...</TouchableHighlight>
// 新代码
<View onPress={handlePress}>...</View>
```

**操作步骤**：
1. 从 import 语句中移除 `TouchableOpacity`/`TouchableWithoutFeedback`/`TouchableHighlight`
2. 在 JSX 中将这些组件标签替换为 `View`
3. 保留 `onPress` prop（`View` 在 `@kds/web` 中支持 `onPress`）
4. 根据需要添加 `activeOpacity` prop 控制点击反馈

---

## 7. 类型迁移（最关键改动）

### 7a. 推荐方式：从 `@kds/web` Props 派生

**不要**从 `react-native` 或 `@types/react-native` 导入样式类型。推荐从 `@kds/web` 的组件 Props 中派生：

```ts
import type { ViewProps, TextProps } from '@kds/web';

type MyProps = {
  containerStyle?: ViewProps['style'];    // 替代 ViewStyle
  textStyle?: TextProps['style'];         // 替代 TextStyle
};
```

**原因**：
- `@kds/web` 的类型在所有端（Web/KRN/TK）保持一致
- 避免了 `@types/react-native` 与 `@kds/web` 的类型冲突
- 无需额外安装类型包
- **特别注意**：即使 `@kds/web` 自身也导出了 `TextStyle`/`ViewStyle` 类型，**也不要直接使用**。原因是 `@kds/web` 导出的 `TextStyle` 是从 `react-native` 原样 re-export 的，但 KDS 的 `<Text>` 组件实际期望的 style 类型是内部定义的完全不同的 `TextStyle`。两者在多个属性上不兼容（如 `fontVariant` 类型不同——数组 vs 字符串联合类型；`flex` 属性有无等）。因此必须使用 `TextProps['style']` 从 Props 派生，才能获得与组件实际匹配的类型。

### 7b. KDS TextStyle 限制

`@kds/web` 的 `TextProps['style']` **不包含** flex 布局属性（如 `alignSelf`、`transform` 等）。如果需要在 Text 的 style 中使用这些属性，需要 `as any`：

```ts
<Text
  style={[{
    color: '#FE3666',
    alignSelf: 'center',
    transform: [{ translateY: rem(-2) }],
  } as any, separatorStyle]}
>
```

### 7c. 类型迁移速查表

| 旧类型（react-native） | 推荐替代方式 | 说明 |
|-----------------------|-------------|------|
| `ViewStyle` | `ViewProps['style']` | 从 `@kds/web` 导入 `ViewProps` |
| `TextStyle` | `TextProps['style']` | 从 `@kds/web` 导入 `TextProps` |
| `ImageStyle` | `ImageProps['style']` | 从 `@kds/web` 导入 `ImageProps` |
| `StyleProp<ViewStyle>` | `ViewProps['style']` | 已包含 `StyleProp` 包装 |
| `StyleProp<TextStyle>` | `TextProps['style']` | 已包含 `StyleProp` 包装 |
| `GestureResponderEvent` | `Parameters<NonNullable<ViewProps['onPress']>>[0]` | 从 `onPress` 回调精确推导，避免使用 `any` |
| `LayoutChangeEvent` | `Parameters<NonNullable<ViewProps['onLayout']>>[0]` | 从 `onLayout` 回调精确推导，避免使用 `any` |

> **类型推导原则**：优先从对应 Props 的回调参数中精确推导事件类型，仅在无法推导时才使用 `any`。示例：
> ```ts
> import type { ViewProps } from '@kds/web';
> type LayoutEvent = Parameters<NonNullable<ViewProps['onLayout']>>[0];
> type PressEvent = Parameters<NonNullable<ViewProps['onPress']>>[0];
> ```

### 7d. 常见 TS 错误及解决方案

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `Type 'ViewStyle' is not assignable to...` | `@types/react-native` 与 `@kds/web` 类型冲突 | 删除 `@types/react-native`，使用 `ViewProps['style']` |
| `Property 'alignSelf' does not exist on type...` | KDS TextStyle 不含 flex 属性 | 给样式对象加 `as any` |
| `Parsing error: '>' expected` 或语法错误 | TypeScript 版本过低，无法解析 inline type import | 升级 `typescript` 到 `^5.0.0` |
| `Cannot find name 'Tachikoma'` | TK 全局变量无类型声明 | 添加 `// @ts-ignore` |
| `Argument of type '{ pageProps: any; }' is not assignable to parameter of type 'IGlobalConfig'` | `setGlobalConfig` 要求 `logger` 字段 | 添加 `// @ts-ignore` |
| `Type 'unknown' is not assignable to type 'string'` | `getUrlParam()` 返回 `unknown` | 添加 `as string` 类型断言 |

---

## 8. Demo 文件修改

### 8a. `demos/demo1.tsx`

将 import 从旧包改为新包即可：

```ts
// 1.0
import { View } from 'react-native';
// 2.0
import { View } from '@kds/web';
```

### 8b. `demos/index.tsx`（KRN 入口）

```ts
// 1.0
import { setPageProps, getUrlParam, env } from '@es/rtx-api';
// 2.0
import { setGlobalConfig, getUrlParam } from '@kds/web-api';
```

**注意**：
- `setGlobalConfig({ pageProps: props })` 需要 `// @ts-ignore`，因为 `IGlobalConfig` 要求 `logger` 字段
- `getUrlParam('pageName')` 返回 `unknown`，需要 `as string` 断言
- **1.0-B 特殊处理**：旧 `setPageProps(props)` 要改为 `setGlobalConfig({ pageProps: props })`，API 签名不同
- **1.0-B 特殊处理**：旧文件中 `env.isWeb` 判断下的 Web 初始化代码应移至单独的 `demos/index.web.tsx` 文件

```ts
// @ts-ignore
setGlobalConfig({ pageProps: props }); // 必须

const pageName = getUrlParam('pageName') as string;
```

### 8c. `demos/index.web.tsx`（Web 入口）

同样修改 import 路径，并处理上述两个类型问题。

> **1.0-B 注意**：原来的 `demos/index.tsx` 可能同时包含 KRN 和 Web 初始化代码（通过 `env.isWeb` 判断）。迁移到 2.0 后，需要将 Web 初始化代码拆分到独立的 `demos/index.web.tsx` 文件。具体来说：
> 1. 在 `demos/index.tsx` 中删除 `if (env.isWeb) { ... }` 分支及相关的 `env` 导入
> 2. 将该分支中的 Web 初始化逻辑（如 `createRoot`、DOM 挂载等）移至 `demos/index.web.tsx`
> 3. KRN 端代码保留在 `demos/index.tsx` 中

---

## 9. Bridge 调用迁移（1.0-B 关键改动）

1.0-B 项目中使用 `@es/rtx-api` 的 `bridge` 对象进行端能力调用，2.0 中需要替换为 `@kds/web-api`（或 `@kds/web`）的 `invoke` 函数。**不仅包名变了，API 调用签名也完全不同**：

### 9a. `bridge.canIUse` → `invoke('tool.canIUse', ...)`

```typescript
// 1.0-B 旧代码
import { bridge } from '@es/rtx-api';
bridge.canIUse('cdn', 'getCdnHost').then((supported) => {
  if (supported) { /* ... */ }
});

// 2.0 新代码
import { invoke } from '@kds/web-api';
invoke('tool.canIUse', { namespace: 'cdn', name: 'getCdnHost' }).then((supported) => {
  if (supported) { /* ... */ }
});
```

### 9b. `bridge.invoke` → `invoke('ns.method', params)`

```typescript
// 1.0-B 旧代码
import { bridge } from '@es/rtx-api';
bridge.invoke('platform', 'getSystemInfo', {}).then((info) => { /* ... */ });

// 2.0 新代码
import { invoke } from '@kds/web-api';
invoke('platform.getSystemInfo', {}).then((info) => { /* ... */ });
```

### 9c. 转换规则总结

| 1.0-B（bridge） | 2.0（invoke） |
|----------------|--------------|
| `bridge.canIUse(namespace, name)` | `invoke('tool.canIUse', { namespace, name })` |
| `bridge.invoke(namespace, method, params)` | `invoke('namespace.method', params)` |

> **注意**：对高频 bridge 调用（如图片域名 CDN 逃生），保留 `canIUse` 前置判断是合理的——先判断端能力是否可用，再执行 invoke。

---

## 10. 组件库日志处理（debugLogger）

### 业务工程 vs npm 组件的区别

工程迁移指南推荐使用 `@/weblogger` 路径来替代日志功能，但这**仅适用于业务工程**。对于 **npm 组件库**（如 `@es/kprom-*`），组件不知道宿主工程的 weblogger 配置，因此不能引入对宿主工程日志系统的路径依赖。

### npm 组件库的 debugLogger 处理策略

```typescript
// ❌ 错误：组件库不应依赖宿主工程的日志路径
import { debugLogger } from '@/weblogger';

// ✅ 正确：方案 A — 非关键日志直接移除
// 删除 debugLogger 的 import 和所有调用

// ✅ 正确：方案 B — 改为 console.warn（仅 dev 环境关键日志）
if (__DEV__) {
  console.warn('[kprom-xxx] some important debug info');
}
```

**决策标准**：
- 如果日志仅用于开发调试（如打印 props、状态变化），**直接移除**
- 如果日志用于记录关键错误或降级信息，**改为 `console.warn`**
- **不要**在组件库中引入对 `@/weblogger`、`@ks/weblogger` 等宿主工程路径的依赖
