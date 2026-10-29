---
name: frontend-api-cancel
description: AbortController、竞态取消和取消请求的错误处理
paths:
  - frontend/src/api/
  - frontend/src/hooks/
  - frontend/src/pages/
---

# API 取消请求

## 取消请求

取消请求统一使用 `AbortController`，禁止使用 Axios `CancelToken`。

```ts
const controller = new AbortController();

const products = await getProductList(
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
