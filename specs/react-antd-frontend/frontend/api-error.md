---
name: frontend-api-error
description: API 错误模型、统一响应结构和错误处理规则
paths:
  - frontend/src/api/error.ts
  - frontend/src/constants/http.ts
  - frontend/src/types/api.ts
---

# API 错误与响应

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

- API 函数返回后端数据或抛出 `ApiError`，不把 AxiosError 直接泄漏到 UI，也不做二次业务加工。
- 不在 API 函数里弹 toast、跳转路由或写 store；错误提示由拦截器统一处理，成功后的业务动作由调用方在 `await` 接口后决定。
- 全局响应码放 `src/constants/http.ts`，业务模块错误码可放领域目录或 `src/constants/enums.ts`。
- 业务枚举必须使用 `as const` 常量对象派生 union 类型；禁止 `enum`，禁止 API 类型和运行时值重复维护。
- API 参数和响应中的状态、来源、类型码必须引用 `src/constants/`，不得硬编码字面量。
- 401 处理必须防抖，避免并发请求同时触发多次跳转。
- 静默刷新 token 的逻辑放在拦截器或专用 auth API 中，并保证并发请求只刷新一次。
- 上传、下载、长轮询可以扩展独立 client 方法或专用实例，但必须复用错误模型和取消判断。
