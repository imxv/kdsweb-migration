<h1 align="center">KDS-Web Migration Skill</h1>

<p align="center">
  <img src="https://photo.459122.xyz/i/db16ea0b31856af3946fb6714f697f04.png" alt="KDS-Web Migration" />
</p>

<p align="center">
  <a href="./README.md">English</a> / 中文
</p>

---

> 将 KDS / KProM 组件从一码多投 1.0（RTX）迁移到 2.0（`@kds/web` 系列）的Skill。

## 快速安装

```bash
npx skills add https://github.com/imxv/kdsweb-migration
```

> [!TIP]
> 更新 Skill 到最新版本：
>
> ```bash
> npx skills update
> ```

## 概述

为 AI 提供了结构化的迁移指南，用于将基于 `react-native` + `@es/rtx-api` 的一码多投 1.0 组件，自动化迁移至基于 `@kds/web` + `@kds/web-api` 的 2.0 架构。

### 迁移路径

```
react-native / @es/rtx-api  →  @kds/web / @kds/web-api
```

### 核心能力

| 能力 | 说明 |
|------|------|
| **智能版本检测** | 自动分析 `package.json` 和源码 import，判定项目当前版本（1.0 / 2.0 / 混合状态） |
| **包名与 import 替换** | `react-native` UI 组件 → `@kds/web`，`@es/rtx-api` API → `@kds/web-api` |
| **Touchable 组件迁移** | `TouchableOpacity` / `TouchableWithoutFeedback` → `View` + `onPress` |
| **不支持属性处理** | 不直接删除不支持的属性，通过 `nativeProps` 机制透传平台特有属性 |
| **类型系统迁移** | `ViewStyle` / `TextStyle` → `ViewProps['style']` / `TextProps['style']` Props 派生 |
| **API 签名适配** | `setPageProps()` → `setGlobalConfig()`，删除 `setRemBaseLine` |
| **Bridge 调用迁移** | `bridge.canIUse/invoke` → `invoke('ns.method', params)` 统一调用方式 |
| **日志处理** | 组件库中的 `debugLogger` 移除或改为 `console.warn` |
| **TK（Tachikoma）端支持** | 可选，为组件添加 TK 端运行支持 |
| **配置与依赖管理** | 自动更新 `package.json`、`tsconfig.json`、`babel.config.js`、`kds.config.js` 等 |

## 项目结构

```
kdsweb-migration/
├── SKILL.md                              # 主指南：完整的 10 步迁移流程
└── references/
    ├── dependency-changes.md             # 依赖变更详情（新增/删除/更新）
    ├── source-code-changes.md            # 源码变更详情（import、类型、API、Bridge）
    ├── config-changes.md                 # 配置文件变更详情（demo、kds.config、tsconfig 等）
    └── tk-support.md                     # TK 端支持文件模板与说明
```

## 迁移流程概览

完整流程包含 **10 个步骤**，每个步骤均在 [SKILL.md](./SKILL.md) 中详细说明：

1. **版本检测与项目分析** — 判定当前版本，避免误操作
2. **升级 TypeScript** — 前置条件，`@kds/web` 要求 TS ≥ 5.0
3. **更新依赖** — 新增 `@kds/web*`，移除旧依赖，更新 peerDependencies
4. **修改源码** — import 替换、nativeProps 透传、类型迁移、Bridge 迁移、babel 配置
5. **修改 Demo 文件** — 更新 demo 入口文件的 import 和类型标注
6. **修改配置文件** — `kds.config.js`、`config/default.js`、`tsconfig.json`
7. **创建 TK 支持文件**（可选） — `index.tk.ts`、`index.entry.ts`、`tsconfig.tk.json`
8. **处理遗留文件** — `src/browser/` 目录、`krn.config.json`
9. **安装依赖** — `yarn install`
10. **验证** — `jia build`（核心）、`jia dev`、`krn start`、TK 验证

## 使用方式

### 触发条件

当用户提出以下类型的请求时，AI 助手会自动调用本 Skill：

- *"帮我把这个组件迁移到 2.0"*
- *"迁移 KDS 组件到 @kds/web"*
- *"升级一码多投版本"*

### 前置要求

- 本地已安装 `yarn`
- 已配置 `jia` CLI 工具（`@es/jia-config`）

## 常见问题

| 问题 | 解决方案 |
|------|----------|
| TS 语法解析错误 | 升级 `typescript` 到 `^5.0.0` |
| `ViewStyle` 类型冲突 | 删除 `@types/react-native`，使用 `Props['style']` 派生 |
| `alignSelf` 等属性不存在于 TextStyle | 给样式对象加 `as any` |
| `setGlobalConfig` 类型不匹配 | 添加 `// @ts-ignore` |
| `getUrlParam()` 返回 `unknown` | 添加 `as string` 类型断言 |
| `Cannot find name 'Tachikoma'` | `[TK]` 添加 `// @ts-ignore` |
| `src/browser/` 编译错误 | 在 `tsconfig.json` 的 `exclude` 中添加 `"src/browser/**"` |
| `jia build` 首次失败 | 直接重试即可（暂时性缓存问题） |
| TK demo 白屏或运行时错误 | `[TK]` 检查 TK 支持文件是否正确创建 |
| `no exported member 'FlatList'` | `FlatList`/`SectionList` 在 @kds/web 中不存在，保留从 `react-native` 导入 |
| `no exported member 'debugLogger'` | `debugLogger` 在 @kds/web-api 中不存在，直接删除 import 和调用代码 |
| `NativeEventEmitter`/`NativeModules` 未迁移 | 需要从 `react-native` 替换为 `@kds/web`（@kds/web 已重新导出） |
| 不支持的属性被直接删除 | 使用 `nativeProps.rn` 透传，或用统一替代属性（如 `showScrollIndicator`） |

更多问题排查详见 [SKILL.md](./SKILL.md#常见问题排查)。

## KProM 组件迁移快速检查清单

迁移 `@es/kprom-*` 组件时，按以下清单逐项确认：

1. **源码 import**：`react-native` UI 组件 → `@kds/web`（包括 `NativeEventEmitter`、`NativeModules` 等工具 API）
2. **不存在的导出**：`FlatList`/`SectionList` 保留从 `react-native` 导入；`debugLogger` 直接删除
3. **Touchable 组件**：`TouchableOpacity`/`TouchableWithoutFeedback` → `View` + `onPress`
4. **不支持属性处理**：**绝对不能直接删除**，有替代的用替代属性，无替代的移入 `nativeProps.rn`
5. **类型**：不要无脑替换为 `any`，用 `Props['style']` 和 `Parameters<>` 精确推导
6. **API**：`@es/rtx-api` → `@kds/web-api`；`setPageProps()` → `setGlobalConfig()`；删除 `setRemBaseLine`
7. **Bridge**：`bridge.canIUse/invoke` → `invoke('ns.method', params)`
8. **日志**：组件库中的 `debugLogger` 移除或改为 `console.warn`
9. **package.json**：版本号主版本迭代、peerDeps 更新、删除 `@types/react-native`、添加 `typescript: ^5.0.0`
10. **`[TK]` TK 支持**（可选）：简单组件靠编译转换；复杂组件需手写 `.tk.tsx`
11. **Monorepo 首次迁移**：根目录 `yarn add -W @kds/web @kds/web-api`

## 兼容性说明

迁移后的 2.0 组件**可以在 1.0 工程中继续使用**（需宿主工程安装 `@kds/web >= 1.1.5` 并配置 adapter 插件），因此组件可以先行迁移，无需等待消费方工程同步升级。
