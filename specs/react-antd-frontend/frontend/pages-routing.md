---
name: frontend-pages-routing
description: React Router 配置、Route Registry、id、getFullPath、守卫和懒加载
paths:
  - frontend/src/pages/
  - frontend/src/router/
  - frontend/src/layouts/
  - frontend/src/App.tsx
  - frontend/src/main.tsx
---

# 页面与路由

## 页面目录

```text
src/pages/Home/
├── components/
├── hooks/useHomeData.ts
├── index.tsx
└── index.less

src/pages/ProductList/
├── constants.ts
├── hooks/useProductFilters.ts
├── index.tsx
└── index.less
```

规则：

- 页面入口是 `index.tsx`，必须默认导出页面组件。
- 页面私有组件、hook、常量放页面目录；其他页面禁止 import 这些私有实现。
- 页面只做组合、页面级状态和权限入口，不写复杂展示细节。

## 页面样式

- 页面使用 `index.less`，并在页面入口直接 `import './index.less'`。
- 根类名使用 `<kebab-case-page>-page`；所有选择器嵌套在唯一根类名内，禁止 CSS Modules 和 BEM。

## 路由目录

```text
src/router/
├── routes.tsx
├── routeMap.ts
├── guards.tsx
└── lazyLoad.tsx
```

- `routes.tsx` 是唯一 React Router 路由树和 Route Registry 来源。
- `routeMap.ts` 只包含 `AnyRoute`、`getFullPath` 和 `getRouteMap` 工具。
- 不再维护 `modules/**` 或重复的路径常量表。

## 懒加载

```tsx
// src/router/lazyLoad.tsx
import { Suspense, type ReactNode } from 'react';
import { PageLoading } from '@/components/ui/PageLoading';

export default function lazyLoad(children: ReactNode) {
  return <Suspense fallback={<PageLoading />}>{children}</Suspense>;
}
```

规则：

- 路由页面默认 `lazy(() => import(...))`。
- 页面模块必须默认导出组件。
- `React.lazy` / `import()` 只允许出现在 `src/router/routes.tsx`。
- fallback 统一使用 `PageLoading`。
- 首屏关键页面可根据性能数据改成 eager。

## 路由定义与 Route Map

完整定义见 [router-registry.md](./router-registry.md)。路由使用全局唯一 `id`，跳转使用 `getFullPath()`：

```tsx
// src/router/routes.tsx
const routes = [
  {
    id: 'app',
    path: '/app',
    element: <AppLayout />,
    children: [
      { id: 'home', index: true, Component: HomePage },
      {
        id: 'productList',
        path: '/product',
        Component: ProductListPage,
        handle: {
          title: '商品列表',
          requiresAuth: true,
        },
      },
      {
        id: 'productDetail',
        path: '/product/:productId',
        Component: ProductDetailPage,
        handle: {
          title: '商品详情',
          requiresAuth: true,
          permissions: ['product:read'],
        },
      },
    ],
  },
] as const satisfies RouteObject[];

export const routeMap = getRouteMap(routes);
export const router = createBrowserRouter(routes);
```

使用方式：

```ts
routeMap.app.getFullPath();
routeMap.app.home.getFullPath();
routeMap.app.productList.getFullPath();
routeMap.app.productDetail.getFullPath({ productId: '123456' });
```

规则：

- 每个路由必须有全局唯一 `id`。
- `routes` 是唯一 React Router 路由来源；`routeMap` 由同一份路由树生成。
- 跳转使用 `routeMap.<id>.getFullPath()`，不写裸路径。
- 动态参数由 `getFullPath()` 显式传入；参数缺失时保留占位符。
- index 路由使用 `index: true`，route map 中用其 `id` 作为路径段。
- React Router 元信息统一放在 `handle`，不要扩展非标准 `meta` 或 `authRoute`。

## 守卫

```tsx
// src/router/guards.tsx
import { Navigate, Outlet, useLocation, useMatches, type UIMatch } from 'react-router-dom';
import { useAuthStore } from '@/store/auth';

interface AppRouteMeta {
  title?: string;
  requiresAuth?: boolean;
  permissions?: string[];
}

function findGuardMeta(matches: UIMatch[]): AppRouteMeta | undefined {
  return [...matches]
    .reverse()
    .find((match) => Boolean(match.handle))?.handle as AppRouteMeta | undefined;
}

export function RequireAuth() {
  const location = useLocation();
  const matches = useMatches();
  const meta = findGuardMeta(matches);
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated);

  if (meta?.requiresAuth && !isAuthenticated) {
    const from = `${location.pathname}${location.search}`;

    return <Navigate to="/login" replace state={{ from }} />;
  }

  return <Outlet />;
}

export function RequirePermission() {
  const matches = useMatches();
  const meta = findGuardMeta(matches);
  const permissions = useAuthStore((state) => state.permissions);

  const required = meta?.permissions ?? [];
  const hasPermission = required.some((permission) =>
    permissions.includes(permission),
  );

  if (required.length > 0 && !hasPermission) {
    return lazyLoad(<ForbiddenPage />);
  }

  return <Outlet />;
}
```

规则：

- 守卫只读取 auth store / 全局权限 store，不直接调用登录接口。
- 登录跳转必须 `replace`，并保留来源路径和 search。
- 登录成功后优先回到 `state.from`，没有来源时跳转默认页。
- 403 页面不自动重定向；权限初始化失败时按未授权处理。
- `handle.permissions` 表示满足任意一个权限即可访问；需要全部满足时显式扩展守卫。

## App 与 Provider

```tsx
// src/App.tsx
import { RouterProvider } from 'react-router-dom';
import { router } from '@/router/routes';

export default function App() {
  return <RouterProvider router={router} />;
}
```

```tsx
// src/main.tsx
<ConfigProvider theme={lightTheme}>
  <StyleProvider hashPriority="low">
    <NiceModal.Provider>
      <App />
    </NiceModal.Provider>
  </StyleProvider>
</ConfigProvider>
```

规则：

- `App.tsx` 只承载 `RouterProvider`。
- `NiceModal.Provider` 必须位于 `ConfigProvider` / `StyleProvider` 内部。
- 只在 `main.tsx` 挂载一次 Provider。
- 页面、组件、store 不允许再包 `RouterProvider`。

## 标题与菜单

AppLayout 中读取 match handle 维护标题和菜单：

```tsx
const meta = [...useMatches()]
  .reverse()
  .find((match) => Boolean(match.handle))?.handle;

useEffect(() => {
  document.title = meta?.title ? `${meta.title} - Example` : 'Example';
}, [meta?.title]);
```

菜单可复用 `routes`，通过 `handle.hideMenu`、`handle.title` 和 `handle.permissions` 过滤。`document.title` 只在路由 effect 中维护。

## 布局

`src/layouts/AppLayout/index.tsx` 只负责：

- 侧边栏、头部、内容区。
- 主题、语言、用户菜单入口。
- 全局布局 loading / error boundary。
- 渲染 `<Outlet />`。

布局不写领域列表、表单或业务请求。
