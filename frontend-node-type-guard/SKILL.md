---
name: frontend-node-type-guard
description: "纯前端项目（无 SSR/SSG）中的 Node 类型泄漏守卫。USE FOR: React/Vue/Vite 等浏览器项目出现 process/Buffer/node:* 可用、或依赖变更后疑似引入 @types/node 时，快速建立并验证类型守卫；并修复 no-node.guard.ts 在 ESLint 下的 no-unused-expressions/no-unused-vars 问题。DO NOT USE FOR: Next.js/Nuxt/Remix 等 SSR/SSG 项目，或本身包含 Node 运行时代码的场景。"
---

# Frontend Node Type Guard

## 目标

在纯前端 TypeScript 项目中，及时发现 Node 类型（如 `process`、`Buffer`、`node:*`）被意外引入到应用侧类型空间的问题。

## 适用范围

- 纯浏览器端项目（无 SSR/SSG）
- 应用侧类型检查使用独立配置（例如 `tsconfig.app.json`）
- 守卫文件位于应用源码 `include` 范围内

## 常见触发链路

1. `src` 新增 import（特别是 devtools/helper 包）带入依赖声明
2. 新依赖的 `.d.ts` 包含 `/// <reference types="node" />`
3. 新依赖声明中出现 `node:*` 内建模块类型导入
4. 上游依赖间接引用 `@types/node`，并泄漏到 app 类型图

## 实施步骤

1. 在应用源码目录创建守卫文件（建议：`src/types/no-node.guard.ts`）
2. 写入如下内容：

```ts
// 浏览器应用类型空间不应解析 `node:*` 内建模块。
// @ts-expect-error 预期报错：`node:*` 在浏览器应用中不可解析；若可解析，通常是依赖声明（如 `/// <reference types="node" />`）泄漏到 app 类型图。
import type {} from "node:fs";

// 浏览器应用代码不应看到 Node 全局变量 `process`。
// @ts-expect-error 预期报错：前端类型空间不应存在 `process`；若未报错，常见链路是 `src` 新增 import 或新增依赖声明引入了 Node 类型。
export type __NoNodeProcess = typeof process;

// 浏览器应用代码不应看到 Node 全局变量 `Buffer`。
// @ts-expect-error 预期报错：前端类型空间不应存在 `Buffer`；若未报错，常见链路是某个包的 `.d.ts` 间接引用了 `@types/node`。
export type __NoNodeBuffer = typeof Buffer;

export {};
```

3. 运行验证命令：

```bash
pnpm exec tsc -p tsconfig.app.json --noEmit
```

4. 如果项目启用 ESLint，再运行：

```bash
pnpm -C apps/web exec eslint src/types/no-node.guard.ts
```

## ESLint 兼容说明

- 不要写 `process;`、`Buffer;` 这类裸表达式：会触发 `@typescript-eslint/no-unused-expressions`
- 不要写未导出的本地类型别名：可能触发 `@typescript-eslint/no-unused-vars`
- 推荐使用“导出类型别名 + @ts-expect-error”模式，既保留守卫语义，也能通过常见 lint 规则

## 结果判定

- 无输出/通过：守卫生效，当前未检测到 Node 类型泄漏
- 报错 `Unused '@ts-expect-error' directive`：说明对应语句不再报错，Node 类型已泄漏

## 排查建议

1. 回看最近新增的 `src` import 与新依赖
2. 检查错误链路是否落到某依赖 `.d.ts` 或 `vite/dist/node/index.d.ts`
3. 对 devtools 包优先考虑开发态动态导入或 runtime bridge
4. 保持应用侧 `types` 最小化，避免不必要的全局类型注入

## 注意事项

- 本 skill 不适用于 SSR/SSG 项目
- 守卫文件必须纳入应用侧 tsconfig 的 `include`
- 建议将该检查纳入提交前或 CI 流程
