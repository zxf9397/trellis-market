---
name: frontend-cross-cutting-change
description: 跨配置、API、状态、路由和 UI 的联动变更
paths:
  - frontend/src/
---

# 跨层变更指南

新增一个页面或业务能力的推荐顺序：

1. **配置**  
   更新 `.env.example`、`src/config/env.ts`、feature flag 和类型。
2. **API**  
   添加领域 `types.ts`、接口函数、params 类型和响应模型；新增业务枚举时先在 `src/constants/` 定义 `as const` 对象。有文档时补全 JSDoc；无文档时使用 typed API Mock。
3. **状态/Hook**  
   决定服务端状态、客户端状态、权限状态分别放哪里。
4. **路由与弹窗**  
   添加 route `id`、路径、`Component` / `element`、`handle`、守卫和懒加载；业务弹窗/抽屉使用 `NiceModal.show()` 触发，并生成 `routeMap`。
5. **布局/页面**  
   组合布局、页面 hook、business/ui 组件；服务端分页列表使用 ProTable。
6. **成功/错误处理**  
   手动请求把成功分支写在 `useRequest` 的 async service 内，并放在 `await request(...)` 之后；接口错误由统一响应拦截器处理，不在 `useRequest` 里写 `onSuccess` / `onError`。
7. **样式/主题**  
   优先复用 token 和 Ant Design；新增 token 必须说明复用场景。
8. **验证与文档**
   默认执行 `pnpm lint` 和 `pnpm typecheck`，并修复问题；不生成测试文件、不运行 Vitest。测试和构建由开发人员决定；用户要求验证时再更新/执行测试，并更新 `frontend/README.md` 和相关 spec。

## 一致性检查

- 环境变量是否已进入 Zod schema？
- API 错误是否转成 `ApiError`？
- 新路由是否受认证/权限守卫控制？
- 401 后登录态和路由跳转是否一致？
- 菜单、按钮、接口、路由的权限是否同源？
- 新增字典是否放对了常量层？
- 表单是否使用 Ant Design Form，并正确校验、初始化和提交？
- 列表分页是否使用 `pageNo` / `pageSize`，搜索空值是否过滤？
- API 函数第一个参数是否为类型化 `params` 对象？是否只做协议请求和响应透传，业务加工是否放在调用侧？
- 枚举运行时值、API 类型和 UI 元数据是否来自同一个常量来源？
- 生产构建是否意外包含 mock/dev-only 代码？
