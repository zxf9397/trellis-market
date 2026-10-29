---
name: frontend-table-list
description: ProTable 服务端分页列表、搜索列、表格列、分页转换和状态渲染
paths:
  - frontend/src/pages/
  - frontend/src/api/
  - frontend/src/constants/
  - frontend/src/utils/
  - frontend/package.json
---

# 基本表格列表

## 适用范围

服务端分页、可搜索、可筛选的业务列表默认使用 `@ant-design/pro-components` 的 `ProTable`：

```bash
pnpm add @ant-design/pro-components
```

纯静态小表格、前端分页配置表、卡片流不强行使用 ProTable。

## 页面与数据目录

```text
src/pages/GroupPurchaseList/
├── constants.ts
├── index.tsx
└── index.less

src/api/groupPurchase/
├── types.ts
└── index.ts
```

- 列表项、请求参数、接口函数放 `src/api/<domain>/`。
- 跨模块状态枚举和元数据放 `src/constants/enums.ts`。
- 页面私有默认值放页面 `constants.ts`。
- 不为 ProTable 维护 `loading`、`data`、`total`、`pagination` state。
- 页面只组合 ProTable；复杂单元格和动作拆到页面私有组件。

公共分页响应统一使用 `pageNo`：

```ts
export interface PageResult<T> {
  list: T[];
  total: number;
  pageNo: number;
  pageSize: number;
}
```

API 参数和接口函数必须使用领域类型：

```ts
export interface GroupPurchaseListParams {
  quotationId?: string;
  startDate?: string;
  endDate?: string;
  pageNo?: number;
  pageSize?: number;
}

export async function getGroupPurchaseList(
  params: GroupPurchaseListParams,
  { signal }: { signal?: AbortSignal } = {},
) {
  return client.get<PageResult<GroupPurchaseListItem>>('/mc/group-purchases', {
    params,
    signal,
  });
}
```

## 空参数过滤

提交前过滤 `undefined`、`null`、空字符串和空数组：

```ts
export function compactParams<T extends Record<string, unknown>>(params: T): T {
  return Object.fromEntries(
    Object.entries(params).filter(([, value]) => {
      if (value === undefined || value === null || value === '') return false;
      return !(Array.isArray(value) && value.length === 0);
    }),
  ) as T;
}
```

## 状态元数据

状态值、标签、颜色来自 constants，不在列配置里重复硬编码。推荐在 constants 中维护结构化 `as const` 对象，并按 value 派生查找 map：

```ts
export const GroupPurchaseStatus = {
  Draft: { value: 'draft', text: '待提交', color: 'default' },
  Submitted: { value: 'submitted', text: '已报价', color: 'blue' },
} as const;

export type GroupPurchaseStatusValue =
  (typeof GroupPurchaseStatus)[keyof typeof GroupPurchaseStatus]['value'];

export const GroupPurchaseStatusByValue = Object.fromEntries(
  Object.values(GroupPurchaseStatus).map((item) => [item.value, item]),
) as Record<GroupPurchaseStatusValue, (typeof GroupPurchaseStatus)[keyof typeof GroupPurchaseStatus]>;
```

## 列配置

搜索字段和表格字段拆分：

- `searchColumns`：只出现在搜索区，统一 `hideInTable: true`。
- `tableColumns`：只出现在表格，统一 `hideInSearch: true`。
- 最终传给 ProTable：`[...searchColumns, ...tableColumns]`。
- 常规展示列设置 `width` 和 `ellipsis`。
- 状态或字段为空时展示 `-`。
- 操作列 `fixed: 'right'`，并设置足够宽度。
- 日期展示统一使用 `src/utils/format.ts`，不重复手写 `dayjs(...).format(...)`。

搜索日期范围是表单专用字段，request 中必须转换成后端字段：

```ts
if (effectiveTimeRange?.length) {
  requestParams.startDate = effectiveTimeRange[0];
  requestParams.endDate = effectiveTimeRange[1];
}
```

## 页面模板

```tsx
const DEFAULT_PAGE_SIZE = 20;

export default function GroupPurchaseList() {
  const actionRef = useRef<ActionType | undefined>(undefined);
  const abortRef = useRef<AbortController>();
  const navigate = useAppNavigate();

  const searchColumns: ProColumns<GroupPurchaseListItem>[] = [
    /* 只搜索字段 */
  ];

  const tableColumns: ProColumns<GroupPurchaseListItem>[] = [
    /* 表格字段和操作列 */
  ];

  return (
    <div className="group-purchase-list-page">
      <ProTable<GroupPurchaseListItem>
        actionRef={actionRef}
        rowKey="id"
        columns={[...searchColumns, ...tableColumns]}
        cardBordered
        request={async ({ current, pageSize, effectiveTimeRange, ...rest }) => {
          abortRef.current?.abort();
          const controller = new AbortController();
          abortRef.current = controller;

          try {
            const requestParams: GroupPurchaseListParams = {
              ...rest,
              pageNo: current || 1,
              pageSize: pageSize || DEFAULT_PAGE_SIZE,
            };

            if (effectiveTimeRange?.length) {
              requestParams.startDate = effectiveTimeRange[0];
              requestParams.endDate = effectiveTimeRange[1];
            }

            const result = await getGroupPurchaseList(
              compactParams(requestParams),
              { signal: controller.signal },
            );

            return {
              data: result.list || [],
              success: true,
              total: result.total || 0,
            };
          } catch (error) {
            if (!isRequestCancelled(error)) {
              // 统一响应拦截器已处理错误提示。
            }

            return { data: [], success: false, total: 0 };
          }
        }}
        options={false}
        search={{
          optionRender: (_searchConfig, _form, dom) =>
            searchColumns.length > 0 ? dom : [dom[1]],
        }}
        pagination={{
          defaultPageSize: DEFAULT_PAGE_SIZE,
          showQuickJumper: true,
          showSizeChanger: true,
          size: 'default',
        }}
        scroll={{ x: 900 }}
        tableAlertRender={false}
      />
    </div>
  );
}
```

操作列使用 route helper 跳转，不写裸路径；危险动作先使用 Nice Modal 确认，成功后调用 `actionRef.current?.reload()`。

## ProTable 配置约定

- `rowKey` 指向稳定唯一字段。
- `request` 是服务端分页列表唯一数据入口；不要再手动维护列表 state。
- `current` 转成 `pageNo`，`pageSize` 使用统一默认值 `20`。
- 竞态请求使用 `AbortController`；取消请求不作为业务失败、不弹 toast。
- 请求失败返回 `{ data: [], success: false, total: 0 }`；错误提示交给统一响应拦截器。
- `options={false}`；只有明确需要刷新、密度、导出时才开启。
- 没有行选择时设置 `tableAlertRender={false}`。
- `scroll.x` 覆盖主要列宽之和；操作列固定右侧时必须设置。
- 分页默认开启 `showQuickJumper` 和 `showSizeChanger`。
- 不把 ProTable 内部分页状态复制到全局 store。

## 测试要点

- 默认请求 `pageNo=1`、`pageSize=20`。
- 翻页、修改 pageSize、快速跳转参数正确。
- 空筛选不会提交空字符串和空数组。
- 日期范围正确转换为 `startDate` / `endDate`。
- 状态根据 constants 渲染；未知状态展示 `-`。
- 操作列按状态控制展示或禁用。
- 删除、取消成功后表格刷新。
