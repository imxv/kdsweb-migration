# 依赖变更指南

> 本文档中 1.0 的定义见 [SKILL.md](../SKILL.md)。

## 目录

1. [新增 @kds/web* 依赖](#1-新增-kdsweb-依赖) — @es/kope-api 说明
2. [新增 devDependencies](#2-新增-devdependencies) — TypeScript 升级、TK 依赖
3. [删除 devDependencies](#3-删除-devdependencies) — @types/react-native
4. [更新 peerDependencies](#4-更新-peerdependencies)
5. [package.json 新增字段](#5-packagejson-新增字段) — TK 入口、版本号
6. [不变的依赖](#6-不变的依赖)

---

## 1. 新增 @kds/web* 依赖

在 devDependencies 中新增项目源码中实际使用到的 `@kds/web*` 包：

```json
{
  "@kds/web": "^1.1.10",
  "@kds/web-api": "^1.2.0"
}
```

> 仅需添加项目源码中实际使用到的 `@kds/web*` 包。通常至少需要 `@kds/web`（UI 组件）。如果 demo 入口需要 `setGlobalConfig`/`getUrlParam`，还需要 `@kds/web-api`。

### `@es/kope-api` 与 `@kds/web-api` 的关系

`@es/kope-api` 和 `@kds/web-api` 是**同一套能力的不同包名**：
- 工程迁移指南推荐 `@es/kope-api`
- 组件迁移手册推荐 `@kds/web-api`
- 实际项目中两者可能混用（如 TK 端 demo 中使用 `@es/kope-api`，组件源码中使用 `@kds/web-api`）

**建议**：在 kprom 组件迁移中**统一使用 `@kds/web-api`**，保持包名一致性。如果在项目中发现 `@es/kope-api` 的使用，可以保留不改（功能等价），但新增代码应使用 `@kds/web-api`。

---

## 2. 新增 devDependencies

### 2a. TypeScript（最高优先级）

```json
{
  "typescript": "^5.0.0"
}
```

**必须升级**：`@kds/web` 使用了 TypeScript 4.5+ 的 `import type { X } from 'y'` 内联语法（inline type import）。即使设置了 `skipLibCheck: true`，也无法跳过语法解析错误。低版本 TypeScript 会直接报语法错误导致构建失败。

### 2b. `[TK]` TK 端依赖（仅在选择添加 TK 支持时需要）

```json
{
  "@kds/native-js": "^0.2.2",
  "@es/babel-plugin-kope-tk": "^1.0.1-alpha.4",
  "@kds/babel-plugin-jsx-dom-expressions": "^0.2.12",
  "@kds/tachikoma-extra": "^1.2.8",
  "@kds/tachikoma-lib": "^0.9.6"
}
```

详见 [tk-support.md](./tk-support.md)。

---

## 3. 删除 devDependencies

### 3a. `@types/react-native`（关键）

```
删除: "@types/react-native"
```

**原因**：`@types/react-native` 提供的类型定义（如 `ViewStyle`、`TextStyle`）会与 `@kds/web` 导出的同名类型产生冲突。保留它会导致类型不兼容错误。如果项目依赖树中仍有其他包间接依赖了 `@types/react-native`，可在 `tsconfig.json` 中使用 `skipLibCheck: true`（默认已设置）。

---

## 4. 更新 peerDependencies

```json
{
  "peerDependencies": {
    "@kds/web": "*",
    "@kds/web-api": "*",
    "@types/react": "^16.8 || ^17.0 || ^18.0",
    "react": "^16.8 || ^17.0 || ^18.0",
    "react-native": "0.62.2"
  }
}
```

- 新增 `@kds/web: *` 和 `@kds/web-api: *`；移除 `react-native-web`（2.0 由 `@kds/web` 在内部处理 Web 兼容，不再需要消费方提供 `react-native-web`）
- 保留 `react-native`（KRN 端仍需要）、`react`、`@types/react`；其他原有 peerDependencies（如 `@es/rtx-api`、`@kds/kid-ui-plus`）按项目实际需要保留

---

## 5. package.json 新增字段

### 5a. `[TK]` `"tk"` 入口字段（仅在选择添加 TK 支持时需要）

```json
{
  "tk": "./src/index.tk.ts"
}
```

放在 `"react-native"` 字段之后。详见 [tk-support.md](./tk-support.md)。

### 5b. 版本号更新

主版本号需递增。公式：若当前版本为 `X.y.z`，新版本为 `(X+1).0.0`。

例如：`2.0.1` → `3.0.0`

---

## 6. 不变的依赖

以下依赖保持不变，无需修改：

- `react`、`react-dom`、`react-native`
- `@kds/bridge-lite`、`@yoda/bridge`
- `@kds/react-native-svg`
- 测试相关：`jest`、`@testing-library/*`
- babel 相关：`babel-jest`、`babel-plugin-*`
- `dumi`
