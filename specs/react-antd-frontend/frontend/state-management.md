---
name: frontend-state-management
description: 客户端状态、登录态、UI 状态和 store 边界
paths:
  - frontend/src/store/
  - frontend/src/hooks/useAuth.ts
---

# 状态管理

## 先分类，再选容器

| 状态 | 推荐位置 |
| --- | --- |
| 服务端请求结果、分页、loading/error | ahooks `useRequest` / 领域数据 hook |
| 表单输入、tab、临时选择 | 组件或页面 |
| 业务弹窗/业务抽屉的打开、关闭、结果 | `@ebay/nice-modal-react` |
| 主题、语言、全局 collapse | `src/store/ui.ts` |
| 登录用户、token、权限 | `src/store/auth.ts` |
| 只用于父组件组装的数据 | 父组件 props/state |
| 跨多页面共享的业务缓存 | 数据层 hook，必要时再引入全局缓存库 |

不要把 API 响应整包复制进 Zustand；store 保存客户端需要长期共享的最小状态。

## Store 目录

```text
src/store/
├── auth.ts
├── ui.ts
└── index.ts
```

Store 的准入规则、持久化策略、脱敏策略和清理时机统一由 `.trellis/spec/` 维护，不在 `src/store/README.md` 重复。

## Auth store

```ts
interface AuthState {
  token: string | null;
  user: AuthUser | null;
  permissions: string[];
  isAuthenticated: boolean;
  signIn: (payload: SignInResult) => void;
  signOut: () => void;
  hasPermission: (permission: string) => boolean;
}
```

规则：

- `isAuthenticated` 必须由 token/user 派生，不要维护互相矛盾的两个布尔值。
- token 写入 storage 时使用 `src/utils/storage.ts` 收口 key、过期时间和异常处理。
- 权限列表只能来自认证初始化或专用接口，不从前端路由表反推。
- 401、手动退出、账号切换必须统一走 `signOut()`。
- 存储敏感 token 时明确风险；生产环境禁止把长期高权限 token 放 localStorage。

## UI store

只放真正的全局 UI 状态：

- 主题：`light | dark | system`
- 语言
- sidebar collapsed
- 全局 upgrade notice 等展示配置

不要把每个页面 modal open 状态放进来。业务弹窗、登录弹窗、确认框和业务抽屉统一通过 `NiceModal.show()` 触发；如果任意远程入口需要触发它们，由该入口调用 `NiceModal.show()`，而不是把弹窗状态写入 store。

## Action 约定

- action 名称使用动词：`setTheme`、`toggleSidebar`、`signIn`、`signOut`、`reset`。
- 每个切片提供 `reset()`，用于退出登录、切换租户和测试。
- 复杂派生数据优先写 selector，不复制多份可变状态。
- 异步 action 保持在调用方 hook 或业务入口中；store action 默认同步、幂等。
- store 文件不导入组件，不操作 DOM。

## 与路由/API 的协作

- API interceptor 只读取 token，不在里面写复杂登录页逻辑。
- 401 时 interceptor 派发事件或调用 auth action，router guard 负责跳转。
- 登录成功后先写入 auth store，再由路由或页面决定跳转。
- 权限变化必须同步刷新菜单、路由守卫和页面按钮。
