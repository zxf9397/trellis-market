---
name: frontend-api-boundary
description: API params 参数约定和业务加工边界
paths:
  - frontend/src/api/
  - frontend/src/hooks/
  - frontend/src/pages/
  - frontend/src/utils/
---

# API 参数与业务边界

## params 参数约定

API 领域函数的第一个参数必须是类型化的 `params` 对象，禁止使用离散基础类型参数；`options` 只能作为第二个参数。

```ts
// good
const params = {
  productId: '123456',
};

const res = await getProductDetail(params);
```

```ts
// bad
const productId = '123456';

const res = await getProductDetail(productId);
```

规则：

- 每个领域函数必须定义 `XxxParams` 类型，第一个参数命名为 `params`。
- `params` 必须显式传入；不要依赖默认参数让调用方省略。
- 路径参数、query 参数、请求体字段都先收进 `XxxParams`。
- `AbortSignal` 放在第二个 `options` 参数中，不放进离散参数。
- 调用侧优先定义 `const params = { ... }` 后传入 API 函数，提高可读性和复用性。

```ts
// good
export async function getProductDetail(
  params: GetProductDetailParams,
  { signal }: ProductListOptions = {},
) {
  const { productId } = params;

  return client.get<Product>(`/products/${productId}`, { signal });
}
```

```ts
// bad
export async function getProductDetail(
  productId: string,
  signal?: AbortSignal,
) {
  return client.get<Product>(`/products/${productId}`, { signal });
}
```

## 禁止业务加工

`src/api/**` 是 HTTP 边界，不是业务服务层。API 函数只负责发送请求、传递协议参数、接收响应并返回后端数据。

允许放在 API：

- 声明接口 URL、HTTP method。
- 接收和透传后端协议参数。
- 透传 `AbortSignal`。
- 返回 `client` 解包后的后端响应数据。
- 定义与接口文档一致的 DTO 类型。
- 抛出统一拦截器归一化后的 `ApiError`。

禁止放在 API：

- 计算金额、折扣、税费、库存、总价等业务字段。
- 推导 `canEdit`、`canCancel`、`isExpired`、`isFinished` 等业务状态。
- 根据业务状态决定下一步动作、流程分支或权限。
- 对列表做业务过滤、排序、分组、聚合、去重或合并。
- 组合多个接口结果拼装页面模型。
- 添加展示文案、日期格式、金额格式、颜色或排序元数据。
- 写 store、路由跳转、DOM、toast 或组件逻辑。

```ts
// bad
export async function getOrderList() {
  const result = await client.get<PageResult<Order>>('/orders', { params });
  const now = Date.now();

  return result.list.map((item) => ({
    ...item,
    totalAmount: item.price * item.quantity,
    isExpired: new Date(item.expireTime).getTime() < now,
    canCancel: item.status === 'created',
    statusText: item.status === 'created' ? '待支付' : '已完成',
  }));
}
```

业务加工放到调用侧：

- 跨页面复用的纯计算放 `src/utils/`，例如 `calculateOrderAmount()`。
- 页面或组件需要的组合状态放领域 hook / 页面 hook。
- 展示文案和颜色来自 `src/constants/`。
- 跨接口聚合和流程编排放在页面 hook 或领域 hook 中。

```ts
// ok：API 只返回后端数据
export async function getOrderList(
  params: OrderListParams,
  { signal }: OrderListOptions = {},
) {
  return client.get<PageResult<Order>>('/orders', { params, signal });
}
```

```ts
// ok：调用侧加工
function toOrderRows(list: Order[]): OrderRow[] {
  const now = Date.now();

  return list.map((item) => ({
    ...item,
    totalAmount: calculateOrderAmount(item),
    isExpired: isOrderExpired(item, now),
    canCancel: item.status === OrderStatus.Created,
    statusText: OrderStatusByValue[item.status]?.text ?? '-',
  }));
}
```
