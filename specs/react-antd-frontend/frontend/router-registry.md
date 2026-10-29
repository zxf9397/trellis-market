---
name: frontend-router-registry
description: 路由注册表、id、getFullPath、RouteMap 和 React Router 配置
paths:
  - frontend/src/router/routes.tsx
  - frontend/src/router/routeMap.ts
  - frontend/src/router/index.tsx
---

# Route Registry

路由统一使用类型化 Route Registry 定义。路由树是唯一来源；`routeMap` 用于集中跳转，React Router 直接使用同一份 `routes`。

## 类型与工具

```ts
// src/router/routeMap.ts
import type { RouteObject } from 'react-router-dom';

export interface AnyRoute extends Omit<RouteObject, 'id' | 'children'> {
  id: string;
  children?: readonly AnyRoute[];
}

// ---------- 路径拼接 ----------
export const joinPath = (parent: string, segment: string): string => {
  const s = segment.replace(/^\/+/, '');
  if (!s) return parent || '/';
  const p = parent.replace(/\/+$/, '');
  return p ? `${p}/${s}` : `/${s}`;
};

// ---------- 参数替换 ----------
export const fillParams = (
  path: string,
  params?: Record<string, string | number>,
): string => {
  if (!params) return path;

  return path.replace(/:([^/]+)/g, (match, key) => {
    const value = params[key];
    return value != null ? String(value) : match; // 缺失时保留占位符，便于排查
  });
};

// ---------- 类型推导 ----------
export type RouteMap<R extends readonly AnyRoute[]> = {
  [Node in R[number] as Node['id']]: Omit<Node, 'children'> & {
    getFullPath: (params?: Record<string, string | number>) => string;
  } & (Node extends { children: infer C extends readonly AnyRoute[] }
    ? RouteMap<C>
    : unknown);
};

// ---------- 实现 ----------
export function getRouteMap<R extends readonly AnyRoute[]>(
  routes: R,
  parentPath = '',
): RouteMap<R> {
  const map: Record<string, unknown> = {};

  for (const route of routes) {
    // index 路由用 id 当路径段；普通路由用 path
    const segment = route.index ? route.id : (route.path ?? '');
    const fullPath = joinPath(parentPath, segment);

    const node: Record<string, unknown> = {
      ...route,
      getFullPath: (params?: Record<string, string | number>) =>
        fillParams(fullPath, params),
    };

    // 子路由平铺到当前节点
    if (route.children?.length) {
      Object.assign(node, getRouteMap(route.children, fullPath));
    }

    map[route.id] = node;
  }

  return map as RouteMap<R>;
}
```

## 路由定义

```tsx
// src/router/routes.tsx
import { lazy } from 'react';
import { createBrowserRouter, type RouteObject } from 'react-router-dom';
import AppLayout from '@/layouts/AppLayout';
import { getRouteMap } from './routeMap';

const HomePage = lazy(() => import('@/pages/Home'));
const ProductListPage = lazy(() => import('@/pages/ProductList'));
const ProductDetailPage = lazy(() => import('@/pages/ProductDetail'));

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

## 使用方式

```ts
routeMap.app.getFullPath(); // "/app"

routeMap.app.home.getFullPath(); // "/app/home"

routeMap.app.productList.getFullPath(); // "/app/product"

routeMap.app.productDetail.getFullPath({ productId: '123456' });
// "/app/product/123456"
```

## 规则

- 每个路由必须有全局唯一 `id`。
- `routes` 是唯一 React Router 路由来源；`routeMap` 是唯一跳转路径来源。
- `routeMap` 只用于路径跳转，不承载业务加工。
- 动态路径参数由 `getFullPath(params)` 显式传入。
- 参数缺失时保留 `:productId` 占位符，便于排查。
- index 路由在 route map 中使用 `id` 作为路径段。
- 路由元信息放在 React Router 的 `handle` 中，守卫和布局通过 `useMatches()` 读取。
- 新增、修改、删除路由只维护 `routes`。
