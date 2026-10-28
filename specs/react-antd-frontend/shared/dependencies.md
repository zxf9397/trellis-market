---
name: shared-dependencies
description: 前端依赖选择与升级
paths:
  - frontend/package.json
  - frontend/pnpm-lock.yaml
  - frontend/package-lock.json
---

# 依赖管理

## 引入前检查

1. 是否已有等价能力？
2. 是否与 React 18、Vite、TypeScript、Ant Design 兼容？
3. 维护状态、issue 响应、license 是否可接受？
4. 对 bundle 的影响是否可接受？
5. 是否有明确的替代退出方案？

## 约定

- 团队统一使用一个包管理器；锁文件必须提交。
- dependencies 只放运行时依赖，工具放 devDependencies。
- UI 组件优先使用已有 Ant Design；服务端分页表格/列表统一使用 `@ant-design/pro-components` 的 ProTable。
- 时间处理统一使用 `dayjs`，不重复引入 date-fns、moment 等等价库。
- 日期、格式化、请求、状态、表单等核心库不重复引入。
- 组件内数据获取默认使用 `ahooks` 的 `useRequest`；不重复引入等价请求 hook 库，也不自研 `useRequest`。
- 动态 className 统一使用 `classnames`；不引入 `clsx`，不手写数组 filter/join 或模板字符串拼接。
- 业务弹窗、业务抽屉和异步确认框统一使用 `@ebay/nice-modal-react`；不再引入其他 modal manager。
- patch 修复升级可跟随 minor；major 升级必须单独任务和验收。
- 移除功能后同步删除依赖和配置。
