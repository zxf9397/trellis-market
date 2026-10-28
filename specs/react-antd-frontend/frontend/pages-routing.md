---
name: frontend-pages-routing
description: 页面组织、React Router、守卫和懒加载
paths:
  - frontend/src/pages/
  - frontend/src/router/
  - frontend/src/layouts/
  - frontend/src/App.tsx
---

# 页面与路由

## 页面目录

```text
src/pages/Home/
├── components/
│   ├── HeroSection/
│   │   ├── index.tsx
│   │   └── index.less
│   └── SectionCard/
│       ├── index.tsx
│       └── index.less
├── hooks/
│   └── useHomeData.ts
├── index.tsx
└── index.less

src/pages/ProductList/
├── constants.ts
├── hooks/
│   └── useProductFilters.ts
├── index.tsx
└── index.less
```

规则：

- 页面入口是 `index.tsx`，默认导出页面组件。
- 只属于该页面的组件、hook、常量放页面目录。
- 其他页面禁止 import `@/pages/ProductList/constants`、`@/pages/ProductList/hooks/useProductFilters`。
- 两个页面复用 hook 时上移到 `src/hooks/`；复用组件时判断 business/ui。
- 页面只做组合、页面级状态和权限入口，不直接写复杂展示细节。

## 页面样式

- 页面使用普通 Less 文件 `index.less`，并在页面入口 `index.tsx` 直接 `import './index.less'`。
- 页面根类名使用 `<kebab-case-page>-page`，例如 `ProductList` 使用 `.product-list-page`。
- 页面 Less 中所有选择器必须嵌套在唯一根类名内，禁止使用 CSS Modules 和 BEM。

## 路由模块

```text
src/router/
├── modules/
│   ├── home.ts
│   └── product.ts
├── guards.tsx
├── lazyLoad.tsx
└── index.ts
```

路由模块声明路径、meta 和元素：

```ts
// src/router/modules/product.ts
import lazyLoad from '@/router/lazyLoad';
import ProductList from '@/pages/ProductList';

export const productRoutes = [
  {
    path: 'products',
    element: lazyLoad(<ProductList />),
    meta: {
      title: '商品列表',
      requiresAuth: true,
    },
  },
];
```

```ts
// src/router/index.ts
import { createBrowserRouter } from 'react-router-dom';
import AppLayout from '@/layouts/AppLayout';
import { guards } from '@/router/guards';
import { homeRoutes } from './modules/home';
import { productRoutes } from './modules/product';

export const router = createBrowserRouter([
  {
    path: '/',
    element: <AppLayout />,
    children: [
      ...homeRoutes,
      ...guards(productRoutes),
    ],
  },
]);
```

规则：

- 路径字符串集中在 route module 或 `constants/routes.ts`。
- 跳转使用 `ROUTES.products` 或 route helpers，不写裸字符串。
- 每个路由提供 `meta.title`、`meta.requiresAuth` 或 `meta.permissions`。
- 嵌套路由使用相对 path；重定向和 404 必须显式声明。

## 懒加载

`src/router/lazyLoad.tsx` 统一处理 `Suspense`：

```tsx
import { Suspense, type ReactNode } from 'react';
import { PageLoading } from '@/components/ui/PageLoading';

export default function lazyLoad(children: ReactNode) {
  return <Suspense fallback={<PageLoading />}>{children}</Suspense>;
}
```

规则：

- 路由级页面默认懒加载。
- 首屏关键页面可按性能数据决定是否 eager。
- `React.lazy` 和 `import()` 只在 router 或统一 loader 中使用，不在组件里到处写。
- fallback 必须使用统一 loading 组件。

## 守卫

```tsx
// src/router/guards.tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { useAuthStore } from '@/store/auth';

export function RequireAuth() {
  const location = useLocation();
  const isAuthenticated = useAuthStore((state) => state.isAuthenticated);

  if (!isAuthenticated) {
    return <Navigate to="/login" replace state={{ from: location.pathname }} />;
  }

  return <Outlet />;
}

export function guards(routes: RouteObject[]) {
  return [
    {
      element: <RequireAuth />,
      children: routes,
    },
  ];
}
```

规则：

- 认证状态只从 auth store/全局 hook 读取，守卫不直接调登录接口。
- 权限校验必须是纯同步或可等待的规则；复杂权限从后端初始化后写入 store。
- 未登录跳转保留来源路径。
- 403 页面不自动循环跳转。
- 登录成功后优先回到 `state.from`，无来源时跳转默认页。

## 布局

`src/layouts/AppLayout/index.tsx` 只负责：

- 侧边栏、头部、内容区。
- 主题、语言、用户菜单入口。
- 全局布局 loading/error boundary。

布局不写领域列表、表单或业务请求。
