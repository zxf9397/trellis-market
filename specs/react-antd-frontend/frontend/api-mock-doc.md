---
name: frontend-api-mock-doc
description: 无接口文档时的 API Mock、接口文档补齐和 JSDoc 注释
paths:
  - frontend/src/api/**/types.ts
  - frontend/src/api/**/mock.ts
  - frontend/src/api/**/index.ts
---

# API Mock 与接口文档

## 接口文档与类型注释

开发人员提供接口文档后，必须按文档补齐 API 类型和字段注释；禁止只写 `Record<string, unknown>`、`any` 或把文档留在聊天记录里。

```ts
// src/api/order/types.ts

/** 订单导出参数 */
export interface GwOrderExportParams {
  /** 订单编号 */
  order_sn?: string;
  /** 商户订单号 */
  sgcc_pr_id?: string;
  /** 下单开始时间 */
  order_date_start?: string;
  /** 下单结束时间 */
  order_date_end?: string;
  /** 商品物流状态 */
  logistics_status?: string;
  /** 资产签收状态 */
  gw_delivered?: number;
  /** 国网订单号 */
  sgcc_order_no?: string;
}

/** 订单导出结果 */
export interface GwOrderExportResult {
  /** 导出任务 ID */
  task_id: string;
}
```

规则：

- API 目录里的导出 interface、type、函数必须有 JSDoc。
- 接口文档中的每个请求字段和响应字段都要有字段级 JSDoc。
- 字段名、大小写、类型、可选项必须与后端实际协议一致；`snake_case` 保持 `snake_case`，不做前端隐式改名。
- 文档没有说明是否必填时，先按可选字段处理，并加 `TODO(api-doc): 确认是否必填`。
- 字段含义、枚举范围、时间格式、分页字段、错误码有歧义时，先向开发确认；确认结果写进 JSDoc。
- 状态、类型码、字典值确认后必须接入 `src/constants/` 的 `as const` 枚举，不要长期保留裸字符串类型。
- 未知结构使用 `unknown` 并在使用处窄化，禁止 `any`。

API 函数也要写简洁 JSDoc：

```ts
/**
 * 导出国网订单
 *
 * @param params 订单导出筛选参数。
 * @returns 导出任务 ID。
 */
export async function exportGwOrder(
  params: GwOrderExportParams,
) {
  return client.post<GwOrderExportResult, GwOrderExportParams>(
    '/gw/order/export',
    params,
  );
}
```

## 重要业务逻辑注释

除 API 协议外，以下位置必须补充注释：

- 状态流转、审批、售后、支付等业务分支。
- 复杂表单联动和跨字段校验。
- 分页、筛选、导出、批量操作的参数转换。
- 并发请求、竞态取消、token 刷新、401 恢复。
- 性能优化、浏览器兼容、安全兜底等非常规实现。

注释要求详细但简洁，让后续开发不需要反推产品需求也能看懂：

```ts
/**
 * 单飞刷新 token。
 *
 * 并发请求同时收到 401 时，只允许一个请求触发刷新；
 * 其余请求等待刷新结果，避免重复刷新和连续跳转登录页。
 */
```

```ts
// 导出接口只接受后端日期字段；把搜索表单的 range 拆成开始/结束时间。
requestParams.order_date_start = orderDateRange[0];
requestParams.order_date_end = orderDateRange[1];
```

禁止：

- 只写“处理数据”“临时逻辑”“待优化”这类无信息注释。
- 用注释重复 `const a = b.c` 这类代码已经表达的内容。
- 提交注释掉的死代码。
- 提交没有负责人、任务链接或移除条件的 TODO。

## 无接口文档时的 Mock

开发人员没有提供接口文档时，可以根据其口头描述、原型或产品约定先生成接口定义，并用 Mock 数据推进前端开发。

临时 Mock 的唯一入口是领域 API 函数；组件、hook、store 不允许自行构造假数据绕过 API 层。

```ts
// src/api/order/types.ts

// TODO(api-doc): 后端未提供接口文档；字段暂按后端口述和产品原型约定。
/** 订单导出参数 */
export interface GwOrderExportParams {
  /** 订单编号 */
  order_sn?: string;
  /** 商户订单号 */
  sgcc_pr_id?: string;
}

// TODO(api-doc): 后端未提供接口文档；响应结构暂按口头约定。
/** 订单导出结果 */
export interface GwOrderExportResult {
  /** 导出任务 ID */
  task_id: string;
}
```

```ts
// src/api/order/index.ts
import type { GwOrderExportParams, GwOrderExportResult } from './types';

// TODO(api-doc): 后端未提供接口文档；文档提供后删除 mockData、console.log 和本 TODO。
const mockData: GwOrderExportResult = {
  task_id: 'mock-export-task-id',
};

/**
 * 导出国网订单
 *
 * 当前后端未提供接口文档，暂返回 Mock 数据。
 *
 * @param params 订单导出筛选参数。
 * @returns 导出任务 ID。
 */
export async function exportGwOrder(
  params: GwOrderExportParams,
) {
  console.log('订单导出接口 Mock 数据返回', mockData);

  return mockData;
}
```

规则：

- Mock 数据类型必须复用 API `types.ts`，不允许定义无类型的普通对象。
- Mock 数据要贴近真实业务：合法 ID、状态、时间格式、分页字段和边界数据。
- 函数签名必须按预期接口定义，后续接入文档时组件和 hook 不需要改调用方式。
- `console.log` 固定使用 `XXX接口 Mock 数据返回` 格式；`XXX` 替换为中文接口名。
- 临时 Mock 不允许通过环境开关长期维护，也不允许进入生产构建。
- 临时 Mock 必须带 `TODO(api-doc)`，说明等待哪份接口文档。
- 后端文档提供后，必须删除 mock 常量、`console.log`、`TODO(api-doc)` 和未使用的类型导入，并把函数改回真实 `client` 请求。
- 文档接入完成前不要合并到发布分支；如果需要跨任务等待，必须在任务描述中标记“阻塞：等待接口文档”。

## 文档到位后的移除检查

接口文档到位后，按顺序处理：

1. 逐字段校验类型、是否必填、命名、枚举范围和时间格式。
2. 补齐或修正 JSDoc。
3. 把 Mock 返回改为 `client.get` / `client.post`。
4. 删除 `mockData`、`console.log`、`TODO(api-doc)` 和未使用导入。
5. 确认组件和 hook 调用方式不需要变更。
6. 搜索检查：`TODO(api-doc)`、`Mock 数据返回`、领域 `mock` 变量、`src/api/**/mock.ts`。

全部清理后，生产包中不允许残留这个接口的 Mock 代码或日志。
