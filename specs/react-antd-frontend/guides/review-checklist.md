---
name: frontend-review-checklist
description: 前端任务完成前必须逐项确认
paths:
  - frontend/src/
---

# Review Checklist

## 架构

- [ ] 目录和文件命名符合模板。
- [ ] 依赖方向没有反向引用。
- [ ] UI 组件保持零业务。
- [ ] 业务弹窗、业务抽屉和异步确认框使用 `@ebay/nice-modal-react`。
- [ ] 没有 page/store 自维护的业务弹窗开关。
- [ ] 组件内普通数据获取使用 ahooks `useRequest`。
- [ ] 没有手写 loading/data/error state，也没有用 effect 手动编排普通查询。
- [ ] 没有使用 `useRequest` 的 `onSuccess` / `onError`。
- [ ] 手动动作成功后的业务处理写在 `useRequest` async service 内的 `await request(...)` 之后；错误提示不重复处理。
- [ ] 没有页面互相 import 私有实现。
- [ ] 新 alias 同时配置 Vite、TS、ESLint。

## 配置与 API

- [ ] 没有新增未文档化的环境变量。
- [ ] `import.meta.env` 只出现在 `src/config/env.ts`。
- [ ] API 只在 `src/api/**` 调用，并走 `client.get` / `client.post`。
- [ ] 错误统一转成 `ApiError`。
- [ ] API 类型、字段和接口函数已有 JSDoc；字段命名与接口文档一致。
- [ ] 重要业务逻辑有简洁注释。
- [ ] 无接口文档的接口使用 typed API Mock，并带 `TODO(api-doc)` 和约定日志。
- [ ] 已提供接口文档的接口已删除 Mock、`console.log` 和 `TODO(api-doc)`。
- [ ] 服务端分页列表使用 ProTable；`current` 转为 `pageNo`，空参数已过滤。
- [ ] 401 行为与登录态一致。
- [ ] 业务枚举使用 `as const` 对象，没有使用 `enum`。
- [ ] 状态、类型码和字典值没有硬编码，运行时引用常量成员。
- [ ] 展示文案、颜色、排序等元数据来自 constants，不在组件里重复维护。

## UI/UX

- [ ] 所有 `index.less` 使用唯一根类名包裹。
- [ ] 样式通过 `import './index.less'` 导入。
- [ ] 没有 CSS Modules、`styles.xxx`、BEM 或裸顶层业务选择器。
- [ ] loading、empty、error、success 状态完整；ProTable 失败时返回空列表和 `success: false`。
- [ ] 主题、响应式和移动端行为符合要求。
- [ ] 媒体查询使用 `.media-down()` / `.media-up()`，没有裸断点，并保持 PC 优先。
- [ ] 表单使用 Ant Design `Form` / `Form.Item`，类型明确、校验清晰、初始化正确。
- [ ] 提交 loading 中禁止重复提交；失败保留输入，成功后执行重置/刷新。
- [ ] 可访问性检查通过。
- [ ] UI 截图或录屏已确认。

## 状态与副作用

- [ ] store 只保存应全局共享的最小状态。
- [ ] 组件卸载后清理异步副作用。
- [ ] 请求取消、过期响应、重复提交已处理；取消使用 `AbortController` 且不弹错误提示。
- [ ] 权限和登录态变化不会产生不一致 UI。

## 质量

- [ ] `lint`、`typecheck`、`test`、`build` 通过。
- [ ] 关键路径有测试。
- [ ] 没有 secret、无关调试日志、注释掉的死代码。
- [ ] 新依赖必要且锁文件更新。
- [ ] 相关文档和 spec 已更新。
