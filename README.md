<h1 align="center">KDS-Web Migration Skill</h1>

<p align="center">
  <img src="https://photo.459122.xyz/i/db16ea0b31856af3946fb6714f697f04.png" alt="KDS-Web Migration" />
</p>

<p align="center">
  English / <a href="./README.zh-CN.md">中文</a>
</p>

---

> A Skill for migrating KDS / KProM components from Cross-Platform 1.0 (RTX) to 2.0 (`@kds/web` series).

## Quick Install

```bash
npx skills add https://github.com/imxv/kdsweb-migration
```

> [!TIP]
> Update the Skill to the latest version:
>
> ```bash
> npx skills update
> ```

## Overview

Provides AI with a structured migration guide to automatically migrate components built on `react-native` + `@es/rtx-api` (Cross-Platform 1.0) to the `@kds/web` + `@kds/web-api` (2.0) architecture.

### Migration Path

```
react-native / @es/rtx-api  →  @kds/web / @kds/web-api
```

### Core Capabilities

| Capability | Description |
|------------|-------------|
| **Smart Version Detection** | Analyzes `package.json` and source imports to determine project version (1.0 / 2.0 / mixed) |
| **Package & Import Replacement** | `react-native` UI components → `@kds/web`, `@es/rtx-api` APIs → `@kds/web-api` |
| **Touchable Component Migration** | `TouchableOpacity` / `TouchableWithoutFeedback` → `View` + `onPress` |
| **Unsupported Props Handling** | Never deletes unsupported props — uses `nativeProps` mechanism to pass platform-specific attributes |
| **Type System Migration** | `ViewStyle` / `TextStyle` → `ViewProps['style']` / `TextProps['style']` derived from Props |
| **API Signature Adaptation** | `setPageProps()` → `setGlobalConfig()`, removes `setRemBaseLine` |
| **Bridge Call Migration** | `bridge.canIUse/invoke` → `invoke('ns.method', params)` unified calling convention |
| **Logger Handling** | Removes `debugLogger` from component libraries or replaces with `console.warn` |
| **TK (Tachikoma) Support** | Optional — adds TK runtime support for components |
| **Config & Dependency Management** | Auto-updates `package.json`, `tsconfig.json`, `babel.config.js`, `kds.config.js`, etc. |

## Project Structure

```
kdsweb-migration/
├── SKILL.md                              # Main guide: complete 10-step migration workflow
└── references/
    ├── dependency-changes.md             # Dependency changes (add/remove/update)
    ├── source-code-changes.md            # Source code changes (imports, types, APIs, Bridge)
    ├── config-changes.md                 # Config file changes (demo, kds.config, tsconfig, etc.)
    └── tk-support.md                     # TK support file templates and instructions
```

## Migration Workflow

The complete workflow consists of **10 steps**, each detailed in [SKILL.md](./SKILL.md):

1. **Version Detection & Analysis** — Determine current version to prevent misoperations
2. **Upgrade TypeScript** — Prerequisite: `@kds/web` requires TS >= 5.0
3. **Update Dependencies** — Add `@kds/web*`, remove legacy deps, update peerDependencies
4. **Modify Source Code** — Import replacement, nativeProps passthrough, type migration, Bridge migration, babel config
5. **Modify Demo Files** — Update demo entry file imports and type annotations
6. **Modify Config Files** — `kds.config.js`, `config/default.js`, `tsconfig.json`
7. **Create TK Support Files** (optional) — `index.tk.ts`, `index.entry.ts`, `tsconfig.tk.json`
8. **Handle Legacy Files** — `src/browser/` directory, `krn.config.json`
9. **Install Dependencies** — `yarn install`
10. **Verify** — `jia build` (critical), `jia dev`, `krn start`, TK verification

## Usage

### Trigger Conditions

The AI assistant automatically invokes this Skill when users make requests like:

- *"Migrate this component to 2.0"*
- *"Migrate KDS components to @kds/web"*
- *"Upgrade cross-platform version"*

### Prerequisites

- `yarn` installed locally
- `jia` CLI tool configured (`@es/jia-config`)

## FAQ

| Issue | Solution |
|-------|----------|
| TS syntax parsing error | Upgrade `typescript` to `^5.0.0` |
| `ViewStyle` type conflict | Remove `@types/react-native`, use `Props['style']` derivation |
| `alignSelf` not in TextStyle | Add `as any` to the style object |
| `setGlobalConfig` type mismatch | Add `// @ts-ignore` |
| `getUrlParam()` returns `unknown` | Add `as string` type assertion |
| `Cannot find name 'Tachikoma'` | `[TK]` Add `// @ts-ignore` |
| `src/browser/` compile errors | Add `"src/browser/**"` to `tsconfig.json` `exclude` |
| `jia build` fails on first run | Retry directly (temporary cache issue) |
| TK demo blank screen or runtime error | `[TK]` Check TK support files are correctly created |
| `no exported member 'FlatList'` | `FlatList`/`SectionList` don't exist in @kds/web — keep importing from `react-native` |
| `no exported member 'debugLogger'` | `debugLogger` doesn't exist in @kds/web-api — delete the import and all call sites |
| `NativeEventEmitter`/`NativeModules` not migrated | Must be replaced from `react-native` to `@kds/web` (re-exported by @kds/web) |
| Unsupported props deleted | Use `nativeProps.rn` passthrough, or unified alternatives (e.g., `showScrollIndicator`) |

See [SKILL.md](./SKILL.md) for full troubleshooting guide.

## KProM Migration Checklist

When migrating `@es/kprom-*` components, verify each item:

1. **Source imports**: `react-native` UI components → `@kds/web` (including `NativeEventEmitter`, `NativeModules`, etc.)
2. **Non-existent exports**: Keep `FlatList`/`SectionList` from `react-native`; delete `debugLogger` entirely
3. **Touchable components**: `TouchableOpacity`/`TouchableWithoutFeedback` → `View` + `onPress`
4. **Unsupported props**: **Never delete directly** — use replacement props or move to `nativeProps.rn`
5. **Types**: Don't blindly use `any` — derive with `Props['style']` and `Parameters<>`
6. **APIs**: `@es/rtx-api` → `@kds/web-api`; `setPageProps()` → `setGlobalConfig()`; delete `setRemBaseLine`
7. **Bridge**: `bridge.canIUse/invoke` → `invoke('ns.method', params)`
8. **Logging**: Remove `debugLogger` from component libraries or replace with `console.warn`
9. **package.json**: Bump major version, update peerDeps, remove `@types/react-native`, add `typescript: ^5.0.0`
10. **`[TK]` TK Support** (optional): Simple components use compile transform; complex ones need manual `.tk.tsx`
11. **Monorepo first migration**: Run `yarn add -W @kds/web @kds/web-api` in root

## Compatibility

Migrated 2.0 components **can still be used in 1.0 projects** (host project needs `@kds/web >= 1.1.5` with adapter plugin configured). Components can be migrated independently without waiting for consumers to upgrade.
