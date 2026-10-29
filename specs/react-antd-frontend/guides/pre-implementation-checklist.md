---
name: frontend-pre-implementation
description: 写前端代码前必须完成的检查
paths:
  - frontend/src/
---

# 实现前检查

## 1. 定位需求

- 交互、数据、错误、空态、权限分别是什么？
- 是否影响路由、登录态或全局布局？路由 `id`、path 和 `handle` 是否唯一且完整？
- 需要兼容的浏览器、移动端和主题是什么？
- 验收标准能否用 UI 行为描述？

## 2. 先读仓库

- 相同页面/组件/hook 是否已经存在？
- API 模块是否已有接口或类型？
- 常量、枚举、格式化工具是否可复用？业务枚举是否已用 `as const` 对象收口？
- 全局样式、Ant Design token 是否已满足视觉？

## 3. 选择边界

按顺序判断：

1. 页面私有内容放 `src/pages/<Page>/`
2. 多页面业务组件放 `src/components/business/`
3. 纯展示组件放 `src/components/ui/`
4. 请求封装放 `src/api/<domain>/`；业务加工放领域 hook、页面或纯工具函数
5. 数据获取默认使用 ahooks `useRequest`；跨页面 hook 放 `src/hooks/`
6. 全局状态放 `src/store/`

## 4. 设计数据流

- 服务端状态在哪里获取、刷新；普通查询是否已用 ahooks `useRequest`？服务端分页列表是否使用 ProTable？
- 错误是否由统一响应拦截器处理；成功后的业务动作是否写在 `await` 接口之后？
- 状态、类型码和字典值是否引用 constants，没有硬编码？
- 表单值、提交参数和接口协议字段如何转换？API 是否只做协议请求，业务加工是否放在调用侧？
- loading/empty/error/success 如何表现？
- 请求取消是否通过 `AbortController` 传递 signal？过期响应、重复触发和竞态如何处理？
- 是否需要 401、权限、跳转？

## 5. 明确验证

- 默认基础检查使用 `pnpm lint` 和 `pnpm typecheck`，发现类型或语法问题必须修复。
- 用户是否已明确要求 test 或 build 验证？
- 未要求时，默认不生成测试文件、不运行 Vitest、不执行构建；完成后可询问是否继续 test / build 验证。
- 用户要求扩展验证时，test、build 分别会覆盖什么？需要新增哪些测试？
- 是否已有接口文档？无文档时是否按口头约定建立 typed API Mock 并记录 `TODO(api-doc)`？
- 本地如何还原目标环境？
