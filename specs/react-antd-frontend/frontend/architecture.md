---
name: frontend-architecture
description: 前端目录、模块边界和依赖方向
paths:
  - frontend/src/
  - frontend/vite.config.ts
  - frontend/tsconfig.json
  - frontend/tsconfig.app.json
  - frontend/eslint.config.mjs
---

# 目录与架构

## 分层依赖方向

只能按下列箭头方向依赖，禁止反向依赖：

```text
assets
  ↓
constants / types / utils
  ↓
config
  ↓
api
  ↓
store / hooks
  ↓
components/ui
  ↓
components/business
  ↓
layouts / pages
  ↓
router
  ↓
App / main
```

同层文件避免互相缠绕；需要共享时上移到更低层。跨层共享的业务字典放 `src/constants/`，单个页面私有字典放该页面的 `constants.ts`。

## 目录职责

| 路径 | 职责 | 不允许 |
| --- | --- | --- |
| `src/main.tsx` | 挂载 React、Nice Modal Provider、StyleProvider、ConfigProvider、全局样式 | 写业务分支 |
| `src/App.tsx` | 只承载 `RouterProvider` 或应用外壳 | 定义页面 |
| `src/api/` | HTTP、拦截器、错误、按领域的接口模块 | 渲染 UI、操作 DOM |
| `src/components/ui/` | 零业务、可复用、props 驱动 | 依赖 `api/`、`store/`、页面 |
| `src/components/business/` | 有业务含义的复合组件；业务弹窗/抽屉统一使用 Nice Modal | 依赖 `pages/`、`router/`、自维护 `open` 状态 |
| `src/config/` | 环境变量解析、feature flags | 存放可变运行时状态 |
| `src/constants/` | 跨模块 `as const` 枚举、响应码、字典和 UI 元数据 | 使用 `enum`、存放函数副作用或组件样式 |
| `src/hooks/` | 跨页面复用 hooks | 渲染全局布局 |
| `src/layouts/` | 应用框架、导航、内容区域 | 写领域接口调用 |
| `src/mocks/` | 本地开发、测试、联调兜底数据 | 出现在生产入口 |
| `src/pages/` | 路由页面及其私有组件/hooks/constants | 被其他页面 import 私有内容 |
| `src/router/` | 路由表、守卫、lazy 收口 | 直接写页面实现 |
| `src/store/` | 跨页面客户端状态 | 替代服务端缓存或保存全部响应 |
| `src/styles/` | design token、主题、global、antd | 写组件私有样式 |
| `src/types/` | 全局 API 类型、ambient 类型 | 写运行时逻辑 |
| `src/utils/` | 纯函数和通用浏览器封装 | 依赖 React、store、API |

## Alias 约定

`vite.config.ts` 与 `tsconfig.app.json` 至少保持：

```ts
// vite.config.ts
resolve: {
  alias: {
    '@': path.resolve(__dirname, 'src'),
  },
},
```

```json
// tsconfig.app.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

规则：

- `@/` 指向 `src/`。
- 跨目录引用使用 `@/api/user`、`@/components/ui/...`，不写超过一级的 `../../`。
- 静态资源放 `src/assets/`，通过 `@/assets/...` 引用；构建期 public 资源才放 `public/`。
- 新增 alias 必须同步 Vite、TypeScript 和 ESLint boundaries。

## 新增模块判断

按顺序选择放置位置：

1. 只属于一个页面：`src/pages/<Page>/...`
2. 多页面复用且有业务含义：`src/components/business/<Name>/`
3. 无业务含义、只由 props/事件控制：`src/components/ui/<Name>/`
4. 跨页面请求逻辑：`src/api/<domain>/` + ahooks `useRequest`（可封装进 `src/hooks/useXxx.ts`）
5. 跨页面客户端状态：`src/store/<slice>.ts`
6. 跨模块字典：`src/constants/`，并使用 `as const` 常量对象
7. 业务弹窗、业务抽屉、异步确认框：`src/components/business/<Name>/` + `NiceModal.show()`
