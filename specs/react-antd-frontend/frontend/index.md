---
name: frontend-index
description: React + Vite 前端规范导航
---

# 前端规范导航

按修改类型读取对应规范：

| 变更 | 必读 |
| --- | --- |
| 新增目录、模块、alias、构建行为 | [architecture.md](./architecture.md) |
| `.env*`、`VITE_*`、`config.ts`、`public/config.js` | [environment-config.md](./environment-config.md) |
| API、Axios、实例、命名 | [api-layer.md](./api-layer.md) |
| API params、业务加工边界 | [api-boundary.md](./api-boundary.md) |
| API 取消、AbortController | [api-cancel.md](./api-cancel.md) |
| API 错误码、401、响应结构 | [api-error.md](./api-error.md) |
| API Mock、接口文档、JSDoc | [api-mock-doc.md](./api-mock-doc.md) |
| 枚举、业务字典、常量对象 | [constants-enums.md](./constants-enums.md) |
| UI 组件、业务组件、页面局部组件 | [components.md](./components.md) |
| Ant Design Form、Form.Item、表单提交 | [form.md](./form.md) |
| 业务弹窗、业务抽屉、异步确认框 | [modal-drawer.md](./modal-drawer.md) |
| 页面、路由、守卫、懒加载 | [pages-routing.md](./pages-routing.md) |
| Route Registry、route id、getFullPath | [router-registry.md](./router-registry.md) |
| 全局状态、登录态、UI 状态 | [state-management.md](./state-management.md) |
| ahooks `useRequest`、数据获取 hooks | [hooks-data.md](./hooks-data.md) |
| 服务端分页表格、搜索列表、ProTable | [table-list.md](./table-list.md) |
| Less、主题、Ant Design | [styling-antd.md](./styling-antd.md) |
| 类型、import 边界、ESLint | [typescript-boundaries.md](./typescript-boundaries.md) |
| 测试、格式化、pre-commit、review | [testing-quality.md](./testing-quality.md) |

## 高频强制项

- 禁止在组件或页面直接 `import axios`；所有 HTTP 必须走 `src/api/client.ts` 封装的类型增强 `client.get` / `client.post`。
- 取消请求统一使用 `AbortController`；拦截器对取消请求直接 reject，不弹 toast。
- 禁止使用 `enum`；枚举必须使用 `as const` 对象。
- 禁止硬编码状态、类型码和业务字典值，运行时必须引用常量成员。
- 业务弹窗、业务抽屉和异步确认框必须使用 `@ebay/nice-modal-react` 封装。
- 表单统一使用 Ant Design `Form` / `Form.Item`；提交前校验，loading 中禁止重复提交。
- 每次变更默认执行 `pnpm lint` 和 `pnpm typecheck`；测试、构建由开发人员决定。
- 用户未要求测试时，不生成测试文件、不运行 Vitest。
- 服务端分页业务列表默认使用 ProTable，统一转换 `pageNo` / `pageSize` 并过滤空参数。
- 不使用 `useRequest` 的 `onSuccess` / `onError`；错误由响应拦截器统一处理。
- 禁止绕过 `src/config/env.ts` 直接读取 `import.meta.env`。
- 禁止 `ui/` 组件依赖 API、store、页面或业务模型。
- 禁止页面之间互相 import 私有 `components/`、`hooks/` 或 `constants.ts`。
- 路由使用 `createBrowserRouter`、唯一 route `id`、统一守卫和 Route Registry；跳转使用 `routeMap.<id>.getFullPath()`。
- Route Registry 统一维护 `id`、`path`、`Component` / `element`、`handle` 和 `getFullPath`；禁止裸路径。
- 禁止把密钥、内部签名、长期高权限 token 暴露给 `VITE_*` 或 `public/config.js`。
- 无接口文档时按口头约定生成 typed API Mock，并在文档到位后删除全部 Mock 代码和日志。
- API 类型和字段必须有 JSDoc；重要业务逻辑必须有简洁注释。
- 媒体查询统一使用 `src/styles/mixins.less` 封装的断点 mixin，不手写裸断点。
