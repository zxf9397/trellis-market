---
name: shared-typescript
description: TypeScript 基础规范
paths:
  - frontend/**/*.ts
  - frontend/**/*.tsx
---

# TypeScript 规范

- 开启 strict；不使用 `any` 作为绕过类型检查的工具。
- 外部数据必须经过类型守卫或 Zod 校验。
- 优先 `unknown` + 窄化，不使用 `as` 压制真实错误。
- 导出的函数和对象写显式类型。
- 联合状态使用 discriminated union。
- 命名：类型/组件/枚举常量对象 PascalCase，普通变量函数 camelCase，普通配置常量 SCREAMING_SNAKE_CASE，文件入口 index.tsx，普通模块 camelCase 或 kebab-case，组件文件 PascalCase。
- 不在运行时代码里写仅类型文件的逻辑。
- ambient 声明集中在 `*.d.ts`，新增全局对象必须显式类型。
