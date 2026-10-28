---
name: frontend-api-layer
description: Axios 实例、类型增强 client、取消请求、错误和 401 处理
paths:
  - frontend/src/api/
  - frontend/src/constants/http.ts
  - frontend/src/types/api.ts
---

# API 层

## 目录

```text
src/api/
├── instance.ts
├── client.ts
├── interceptors.ts
├── error.ts
├── user/
│   ├── types.ts
│   └── index.ts
└── products/
    ├── types.ts
    └── index.ts
```

- `instance.ts` 只使用 `axios.create()` 创建并隔离 Axios 实例。
- `client.ts` 只封装类型增强的 `get` / `post`。
- `interceptors.ts` 只注册请求 / 响应拦截器。
- `error.ts` 定义 `ApiError`、取消判断和错误归一化。
- 领域接口放在 `user/index.ts`、`products/index.ts`。
- 领域数据结构放同目录 `types.ts`，公共响应结构放 `src/types/api.ts`。
- API 类型、字段和接口函数必须有 JSDoc；无接口文档时的 Mock 统一见 [api-mock-doc.md](./api-mock-doc.md)。
- `axios` 原始模块只允许出现在 `instance.ts`、`interceptors.ts` 和 `error.ts` 中；用途仅限创建实例、类型和取消判断。
- 组件、hook、store、页面、领域 API 禁止直接 `import axios` 或直接调用 `axios.get` / `axios.post`。

## 隔离实例

```ts
// src/api/instance.ts
import axios from 'axios';
import { env } from '@/config/env';

export const instance = axios.create({
  baseURL: env.apiBaseUrl,
  timeout: 15_000,
});
```

`instance` 只作为底层 Axios 实例，不导出给业务代码。业务代码不能直接 `await instance.get(...)`。

## 类型增强 Client

```ts
// src/api/client.ts
import type { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';
import { instance } from './instance';
import { registerInterceptors } from './interceptors';
import type { ApiResponse } from '@/types/api';

registerInterceptors(instance);

export type RequestConfig = Omit<AxiosRequestConfig, 'url' | 'method'> & {
  signal?: AbortSignal;
};

async function unwrap<TData>(
  request: Promise<AxiosResponse<ApiResponse<TData>>>,
): Promise<TData> {
  const response = await request;
  return response.data.data;
}

export const client = {
  get<TData>(url: string, config?: RequestConfig): Promise<TData> {
    return unwrap<TData>(instance.get<ApiResponse<TData>>(url, config));
  },

  post<TData, TBody = unknown>(
    url: string,
    data?: TBody,
    config?: RequestConfig,
  ): Promise<TData> {
    return unwrap<TData>(instance.post<ApiResponse<TData>>(url, data, config));
  },
};
```

类型增强后的调用直接得到业务数据，不需要每个接口重复解 `AxiosResponse<ApiResponse<T>>`：

```ts
const product = await client.get<Product>(`/products/${productId}`);
const created = await client.post<Product, CreateProductPayload>('/products', payload);
```

规则：

- 所有 HTTP 请求必须通过 `client.get` / `client.post`。
- 领域函数返回业务数据 `TData`，不返回 `AxiosResponse`。
- `ApiResponse` 解包只发生在 client 或拦截器边界，不在组件里重复解包。
- 需要扩展 `put` / `patch` / `delete` / 上传 / 下载时，必须按同一模式封装到 `client.ts`，不允许绕过实例。
- 文件上传可以扩展独立方法，但要复用同一 `instance`、错误模型和取消判断。

## 领域接口

```ts
// src/api/products/types.ts
import type { ProductStatusValue } from '@/constants/enums';
import type { PageResult } from '@/types/api';

export interface Product {
  id: string;
  name: string;
  price: number;
  status: ProductStatusValue;
}

export interface ProductListParams {
  keyword?: string;
  status?: ProductStatusValue;
  pageNo?: number;
  pageSize?: number;
}

export interface ProductListOptions {
  signal?: AbortSignal;
}
```

```ts
// src/api/products/index.ts
import { client } from '@/api/client';
import type { PageResult } from '@/types/api';
import type { Product, ProductListParams, ProductListOptions } from './types';

export async function listProducts(
  params: ProductListParams = {},
  { signal }: ProductListOptions = {},
): Promise<PageResult<Product>> {
  return client.get<PageResult<Product>>('/products', {
    params,
    signal,
  });
}
```

命名：

- 查询使用 `getXxx`、`listXxx`。
- 创建、更新、删除使用 `createXxx`、`updateXxx`、`deleteXxx`。
- 文件上传使用 `uploadXxx`。
- 禁止把组件名写进接口函数名，例如 `fetchHomeProductListForHeroSection`。

## 取消请求

取消请求统一使用 `AbortController`，禁止使用 Axios `CancelToken`。

```ts
const controller = new AbortController();

const products = await listProducts(
  { pageNo: 1, pageSize: 20 },
  { signal: controller.signal },
);

controller.abort();
```

可取消接口的最后一个参数接收 `signal`，并透传给 client：

- 查询函数支持可选 `options.signal`。
- 需要取消竞态请求时，由调用方创建并保存 `AbortController`。
- 新请求发出前取消上一个请求。
- 组件卸载或弹窗关闭时可以取消仍在进行的查询。
- 取消不代表业务失败，不能展示错误提示。

```ts
// src/api/error.ts
import axios from 'axios';

export function isRequestCancelled(error: unknown): boolean {
  return (
    axios.isCancel(error) ||
    (error instanceof DOMException && error.name === 'AbortError') ||
    (error instanceof Error && error.name === 'CanceledError')
  );
}
```

## 拦截器

请求拦截器职责：

- 注入 token。
- 注入 `X-Request-Id` 或 trace id。
- 统一处理 content type。
- 不修改业务参数语义。

响应拦截器职责：

- 第一步判断取消请求：`axios.isCancel(error)` 命中后直接 `Promise.reject(error)`。
- 取消请求不转换成业务失败、不弹 toast、不写全局错误状态。
- 解包 `ApiResponse<T>` 或统一抛出 `ApiError`。
- 映射后端错误码，统一处理网络错误、超时和用户可感知的错误提示。
- 收到 401 时清除登录态并派发全局事件或调用 auth store；不在业务页面各自处理 401。

```ts
// src/api/interceptors.ts
import type { AxiosError, AxiosInstance } from 'axios';
import { isRequestCancelled } from './error';

export function registerInterceptors(instance: AxiosInstance) {
  instance.interceptors.request.use((config) => {
    // 注入 token / trace id / content type
    return config;
  });

  instance.interceptors.response.use(
    (response) => response,
    (error: AxiosError) => {
      if (isRequestCancelled(error)) {
        return Promise.reject(error);
      }

      // 统一归一化业务错误和非取消错误提示。
      return Promise.reject(error);
    },
  );
}
```

调用方不需要判断 Axios 专属取消对象；必须使用 `isRequestCancelled(error)`：

```ts
try {
  await listProducts(filters, { signal: controller.signal });
} catch (error) {
  if (isRequestCancelled(error)) {
    return;
  }

  // 统一响应拦截器已处理错误提示。
}
```

## 错误与响应

```ts
// src/types/api.ts
export interface ApiResponse<T> {
  code: number;
  data: T;
  message: string;
  requestId?: string;
}

export interface PageResult<T> {
  list: T[];
  total: number;
  pageNo: number;
  pageSize: number;
}
```

```ts
// src/api/error.ts
export interface ApiErrorPayload {
  code: number | string;
  message: string;
  status?: number;
  requestId?: string;
  details?: unknown;
}

export class ApiError extends Error {
  readonly code: number | string;
  readonly status?: number;
  readonly requestId?: string;
  readonly details?: unknown;

  constructor(payload: ApiErrorPayload) {
    super(payload.message);
    this.name = 'ApiError';
    this.code = payload.code;
    this.status = payload.status;
    this.requestId = payload.requestId;
    this.details = payload.details;
  }
}
```

规则：

- API 函数返回业务数据或抛出 `ApiError`，不把 AxiosError 直接泄漏到 UI。
- 不在 API 函数里弹 toast、跳转路由或写 store；错误提示由拦截器统一处理，成功后的业务动作由调用方在 `await` 接口后决定。
- 全局响应码放 `src/constants/http.ts`，业务模块错误码可放领域目录或 `src/constants/enums.ts`。
- 业务枚举必须使用 `as const` 常量对象派生 union 类型；禁止 `enum`，禁止 API 类型和运行时值重复维护。
- API 参数和响应中的状态、来源、类型码必须引用 `src/constants/`，不得硬编码字面量。
- 401 处理必须防抖，避免并发请求同时触发多次跳转。
- 静默刷新 token 的逻辑放在拦截器或专用 auth API 中，并保证并发请求只刷新一次。
- 上传、下载、长轮询可以扩展独立 client 方法或专用实例，但必须复用错误模型和取消判断。
