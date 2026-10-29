---
name: frontend-hooks-data
description: ahooks useRequest、数据获取、领域 hooks、mock 与副作用
paths:
  - frontend/src/hooks/
  - frontend/src/mocks/
  - frontend/src/pages/*/hooks/
  - frontend/src/components/
  - frontend/package.json
---

# Hooks 与数据获取

## 默认约定

组件、页面、业务组件中获取数据时，默认使用 `ahooks` 的 `useRequest`：

```bash
pnpm add ahooks
```

- 必须从 `ahooks` 导入：`import { useRequest } from 'ahooks'`。
- 默认自动请求：`manual: false`。
- 按钮、提交、确认、删除等动作使用 `manual: true`。
- 不要手写 `useState` 维护 `loading`、`data`、`error`。
- 不要用 `useEffect` + `setState` 手动编排普通查询。
- 不要重复实现 `src/hooks/useRequest.ts`；也不要创建同名包装器。
- 只有存在真实复用、轮询、分页、并发编排、依赖服务端缓存等必要时，才引入 React Query / SWR / 自研模型，并在任务说明中记录原因。

## 命名与放置

- hook 必须以 `use` 开头。
- 跨页面 hook 放 `src/hooks/useXxx.ts`。
- 页面私有 hook 放 `src/pages/<Page>/hooks/useXxx.ts`。
- 组件私有数据 hook 可放业务组件目录内，例如 `useProducts.ts`。
- 业务 hook 可调用 API/store；通用 hook 不调用 API/store。
- `src/hooks/index.ts` 只做公共导出，避免文件之间循环 import。

## 自动查询示例

```ts
// src/pages/ProductList/hooks/useProductFilters.ts
import { useMemo, useRef, useState } from 'react';
import { useRequest } from 'ahooks';
import { getProductList } from '@/api/products';
import { isRequestCancelled } from '@/api/error';
import type { ProductListParams } from '@/api/products/types';

export function useProductFilters() {
  const [filters, setFilters] = useState<ProductListParams>({
    pageNo: 1,
    pageSize: 20,
  });

  const abortRef = useRef<AbortController>();

  const { data, loading, error, refresh } = useRequest(
    async () => {
      // 发起新请求前取消上一个竞态请求。
      abortRef.current?.abort();

      const controller = new AbortController();
      abortRef.current = controller;

      try {
        return await getProductList(filters, { signal: controller.signal });
      } catch (error) {
        if (isRequestCancelled(error)) {
          return undefined;
        }

        throw error;
      }
    },
    {
      refreshDeps: [filters],
    },
  );

  const hasActiveFilters = useMemo(
    () => Boolean(filters.keyword),
    [filters.keyword],
  );

  return {
    filters,
    setFilters,
    products: data?.list ?? [],
    total: data?.total ?? 0,
    loading,
    error,
    refresh,
    hasActiveFilters,
  };
}
```

规则：

- `refreshDeps` 只放真正影响请求的值。
- 复杂对象作为依赖时，优先用状态 setter 或参数序列化保持语义稳定。
- 页面只消费领域 hook 返回的窄化数据，不直接持有 axios promise。
- 分页、筛选、排序等页面状态放页面 hook；接口定义和响应解包仍由 API 模块处理。
- 多个相关请求只在领域 hook 中编排，不在 JSX 里连续 `.then`。
- 竞态查询使用 `AbortController` 发起新请求前取消旧请求；取消分支使用 `isRequestCancelled(error)` 判断并静默返回。
- `useRequest.cancel()` 只能阻止过期响应写入状态，不能替代 HTTP 层的 `AbortController` 取消。

## 手动动作示例

`useRequest` 不使用 `onSuccess` / `onError`。手动动作的 async service 内先 `await` 接口；只有请求成功后才执行业务动作。接口错误由 `src/api/interceptors.ts` 统一处理，`catch` 只用于阻止未处理 rejection，不重复提示。

```tsx
import { Button, message } from 'antd';
import { useRequest } from 'ahooks';
import { createProduct } from '@/api/products';

export function CreateProductButton() {
  const { loading, run } = useRequest(
    async () => {
      try {
        const params = {
          name: 'Example',
          price: 99,
        };

        await createProduct(params);

        // 请求成功后才执行业务动作。
        message.success('创建成功');
      } catch {
        // 统一响应拦截器已处理错误提示。
        // 这里只阻止未处理 rejection，不重复提示。
      }
    },
    {
      manual: true,
    },
  );

  return (
    <Button type="primary" loading={loading} onClick={() => run()}>
      创建商品
    </Button>
  );
}
```

规则：

- 查询优先使用默认自动模式。
- `create`、`update`、`delete`、提交、确认等动作使用 `manual: true`。
- loading、data、error、run、refresh 直接来自 `ahooks useRequest`。
- 手动动作的业务成功分支写在 `useRequest` 的 async service 内，并放在 `await request(...)` 之后。
- 接口报错由统一响应拦截器处理；`catch` 只用于阻止未处理 rejection 和维持局部 UI 状态，不重复弹错误提示。
- 禁止使用 `useRequest` 的 `onSuccess` 和 `onError`。
- 同一动作重复触发时，通过按钮 loading、防抖、节流或禁用条件阻止。

## 组件内使用边界

- `src/components/ui/` 不允许发请求，也不允许依赖 ahooks 业务请求。
- `src/components/business/` 的业务组件可以自带 `useRequest`，但优先通过 props 或 `NiceModal.show()` 参数接收初始数据。
- 页面组件优先组合领域 hook；直接在页面里写 `useRequest` 适用于简单一次性查询。
- 不要为了“保存服务端数据”把 `useRequest` 的结果复制进 Zustand；只有跨页面长期共享的客户端状态才进 store。
- 弹窗提交成功后，通过回调、事件或领域 hook 的 `refresh()` 刷新列表，不把列表数据整包写进全局 store。

## 错误与状态

- API 层负责把 AxiosError / 后端响应包装成 `ApiError`。
- 接口报错由统一响应拦截器处理；`useRequest` 返回的 `error` 只用于渲染局部错误态，不重复弹全局错误提示。
- `isRequestCancelled(error)` 命中的取消请求不作为业务失败，不展示错误、不弹 toast。
- 普通查询必须覆盖 loading、empty、error、success 四种 UI 状态。
- 自动查询失败时优先提供重试入口；必要时使用 ahooks 的 `retryCount`、`retryInterval`。
- 不在 `useRequest` service 内吞掉错误；需要静默降级时必须记录原因并说明为什么统一错误处理不适用。

## Mock 约定

```text
src/mocks/
├── user.ts
└── products.ts
```

- mock 数据形状必须复用 `src/api/**/types.ts`，避免接口接入后全量修改。无接口文档时的运行时 Mock 规则见 [api-mock-doc.md](./api-mock-doc.md)。
- 本地 mock 通过环境开关或 dev-only 入口启用，不进入生产 bundle。
- 需要 MSW 时，handler 按领域放在 `src/mocks/`，浏览器 worker 只在 development 初始化。
- mock 不要伪造组件内部状态；优先拦截 HTTP 边界，让 `ahooks useRequest` 走真实数据流。
