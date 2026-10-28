---
name: frontend-constants-enums
description: 业务枚举、常量对象、UI 元数据和硬编码治理
paths:
  - frontend/src/constants/
  - frontend/src/api/
  - frontend/src/components/
  - frontend/src/pages/
  - frontend/src/store/
---

# 枚举与常量

## 核心规则

- 禁止使用 TypeScript `enum` 和 `const enum`。
- 枚举、状态、类型码、业务字典必须使用 `as const` 对象。
- 禁止在组件、hook、store、API 参数里硬编码可枚举业务值。
- API 类型和运行时值必须来自同一个常量对象。
- 展示文案、颜色、排序等元数据必须挂在常量对象或派生映射里。
- 常量对象名使用 PascalCase；对应的 union 类型名使用 `Value` 后缀，例如 `UserStatus` 对应 `UserStatusValue`。
- Key 使用 PascalCase。
- Value 是协议值，必须与后端实际传输值完全一致；如果后端是小写就保持小写，如果后端是 PascalCase 就保持 PascalCase。
- 不做隐式大小写转换、隐式数字字符串转换或前后端别名映射。

安装目录约定：

```text
src/constants/
├── enums.ts
├── http.ts
└── index.ts
```

## 简单枚举

```ts
// src/constants/enums.ts
export const UserStatus = {
  Active: 'active',
  Pending: 'pending',
  Disabled: 'disabled',
} as const;

export type UserStatusValue = (typeof UserStatus)[keyof typeof UserStatus];
// => 'active' | 'pending' | 'disabled'

export const UserStatuses = Object.values(UserStatus);
```

> 说明：TypeScript 允许 `const UserStatus` 和 `type UserStatus` 使用同名声明；上面的报错通常来自 ESLint `no-redeclare` 或项目内重复定义。为了规避这类工具冲突和阅读歧义，模板统一使用 `UserStatusValue` 这种显式类型名。

使用方式：

```tsx
import { UserStatus, type UserStatusValue } from '@/constants/enums';

interface User {
  id: string;
  status: UserStatusValue;
}

function canLogin(user: User) {
  return user.status !== UserStatus.Disabled;
}
```

不要这样写：

```ts
// 禁止
enum UserStatus {
  Active = 'active',
  Pending = 'pending',
  Disabled = 'disabled',
}

// 禁止重复定义运行时值和类型
const USER_STATUS = { ACTIVE: 'active' } as const;
type UserStatusValue = 'active' | 'pending' | 'disabled';
```

## 携带 UI 元数据的枚举

当枚举需要中文标签、Ant Design 颜色、排序、icon 等信息时，使用结构化 `as const` 对象：

```ts
// src/constants/enums.ts
export const OrderStatus = {
  Pending: {
    value: 'pending',
    label: '待处理',
    color: 'gold',
    order: 1,
  },
  Shipped: {
    value: 'shipped',
    label: '已发货',
    color: 'blue',
    order: 2,
  },
  Completed: {
    value: 'completed',
    label: '已完成',
    color: 'green',
    order: 3,
  },
} as const;

export type OrderStatusItem = (typeof OrderStatus)[keyof typeof OrderStatus];

export type OrderStatusValue = OrderStatusItem['value'];
// => 'pending' | 'shipped' | 'completed'

export type OrderStatusKey = keyof typeof OrderStatus;
// => 'Pending' | 'Shipped' | 'Completed'

export const OrderStatusValues = Object.values(OrderStatus).map(
  (item) => item.value,
);

export const OrderStatusLabel = Object.values(OrderStatus).reduce<
  Record<OrderStatusValue, string>
>((result, item) => {
  result[item.value] = item.label;
  return result;
}, {} as Record<OrderStatusValue, string>);

export const OrderStatusItems = Object.values(OrderStatus)
  .slice()
  .sort((a, b) => a.order - b.order);
```

使用方式：

```tsx
import { OrderStatus, OrderStatusItems } from '@/constants/enums';
import type { OrderStatusValue } from '@/constants/enums';

interface OrderStatusTagProps {
  status: OrderStatusValue;
}

export function OrderStatusTag({ status }: OrderStatusTagProps) {
  const item = OrderStatusItems.find((option) => option.value === status);

  return <Tag color={item?.color}>{item?.label}</Tag>;
}
```

查找规则：

- 值到元数据的快速查找优先从 `OrderStatusItems` 派生。
- 需要稳定的 `Record` 映射时，可以基于源对象生成。
- 禁止在组件里重复写 `pending => 待处理` 这类映射。
- 禁止让同一组枚举在 `enums.ts`、API types 和组件 constants 里重复定义。

## API 类型与常量

```ts
// src/api/user/types.ts
import type { UserStatusValue } from '@/constants/enums';

export interface User {
  id: string;
  status: UserStatusValue;
}

export interface UpdateUserStatusParams {
  userId: string;
  status: UserStatusValue;
}
```

```ts
// src/api/user/index.ts
import { client } from '@/api/client';
import { UserStatus } from '@/constants/enums';
import type { User } from './types';

export async function getDisabledUsers() {
  return client.get<User[]>('/users', {
    params: {
      status: UserStatus.Disabled,
    },
  });
}
```

规则：

- 响应字段、请求参数、URL query、表单值、路由查询中的枚举值必须使用对应 union 类型。
- 运行时比较必须引用常量成员，例如 `UserStatus.Active`。
- 只有后端协议没有对应枚举语义的普通字面量才允许局部使用。
- 一个字面量如果被第二次使用，或表达状态/来源/类型/级别，就必须上移到 constants。

## 页面私有字典

只属于单个页面的展示字典可以放页面目录：

```ts
// src/pages/ProductList/constants.ts
import type { Product } from '@/api/products/types';

export const PRODUCT_TABLE_COLUMNS = [
  'name',
  'price',
  'status',
] as const satisfies readonly (keyof Product)[];
```

规则：

- 页面私有常量只能被当前页面引用。
- 第二个页面需要复用时，上移到 `src/constants/`。
- 页面私有常量不允许覆盖或转换跨模块业务枚举的 value。
