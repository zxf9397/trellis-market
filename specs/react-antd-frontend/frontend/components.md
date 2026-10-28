---
name: frontend-components
description: UI 组件、业务组件、页面局部组件和文件组织
paths:
  - frontend/src/components/
  - frontend/src/layouts/
---

# 组件规范

## 三类组件

| 类型 | 位置 | 可依赖 | 禁止 |
| --- | --- | --- | --- |
| UI 组件 | `src/components/ui/` | props、类型、常量、`utils/`、样式、Ant Design 基础能力 | `api/`、`store/`、路由、页面、业务组件 |
| 业务组件 | `src/components/business/` | `api/`、`hooks/`、`store/`、UI 组件；弹窗/抽屉类业务组件必须使用 `@ebay/nice-modal-react` | 另一个页面的私有内容、`router/`、布局、自维护业务弹窗开关 |
| 页面局部组件 | `src/pages/<Page>/components/` | 页面 hooks/constants、business/ui 组件 | 被其他页面 import |

判断顺序：先页面局部，再 business，最后 ui。只有第二个真实使用方出现时才提升。

## 目录结构

复用组件目录使用 PascalCase：

```text
src/components/ui/DescriptionDrawer/
├── index.tsx
├── index.less
└── index.test.tsx

src/components/business/AuthModal/
├── components/
│   ├── LoginForm.tsx
│   ├── RegisterForm.tsx
│   └── ForgotPasswordForm.tsx
├── index.tsx
├── index.less
└── index.test.tsx
```

规则：

- 对外入口是 `index.tsx`；业务弹窗/抽屉由调用方通过 `NiceModal.show(...)` 打开，而不是直接设置 `open` props。
- 业务弹窗、业务抽屉和异步确认框必须使用 `@ebay/nice-modal-react`，详见 [modal-drawer.md](./modal-drawer.md)。
- 子组件只被本组件使用时放 `components/`，不导出到全局。
- 样式文件与组件同名目录下的 `index.less`，由 `index.tsx` 直接 `import './index.less'`。
- 每个 `index.less` 必须用组件唯一的根类名包裹；类名由组件目录名转换成 kebab-case，例如 `DescriptionDrawer` 使用 `.description-drawer`。
- 页面根类名使用 `<kebab-case-page>-page`，例如 `ProductList` 使用 `.product-list-page`。
- 组件目录名必须全局唯一；如果出现同名组件，先重命名目录，再创建样式。子组件如 `LoginForm` 有自己的 Less 时使用 `.login-form`，否则复用父组件的根类名。
- 一个目录只表达一个组件，不要把多个无关组件堆在一个目录。
- 需要组合导出时使用命名导出，例如 `export const AuthModalTabs`；默认导出保持组件本体。

## Props 设计

下面的示例只适用于零业务的 UI 组件；业务 Modal/Drawer 不要设计成这种受控形态。

```tsx
import classNames from 'classnames';
import { Drawer, Spin } from 'antd';
import type { Product } from '@/api/products/types';
import './index.less';

interface DescriptionDrawerProps {
  open: boolean;
  title?: React.ReactNode;
  dataSource?: Product;
  loading?: boolean;
  onClose: () => void;
  footer?: React.ReactNode;
}

export default function DescriptionDrawer({
  open,
  title = '详情',
  dataSource,
  loading = false,
  onClose,
  footer,
}: DescriptionDrawerProps) {
  return (
    <Drawer className={classNames('description-drawer', { 'is-loading': loading })} open={open} title={title} onClose={onClose} footer={footer}>
      <Spin spinning={Boolean(loading)}>{dataSource ? renderItems(dataSource) : null}</Spin>
    </Drawer>
  );
}
```

规则：

- 必填数据用必填 props，不要在展示组件里自己 fetch。
- 受控状态由父组件持有；非受控内部状态只在纯交互场景使用。
- 回调以 `on` 开头，动作函数以 `handle` 开头。
- boolean props 优先使用肯定语义：`loading`、`disabled`、`visible`，避免 `notLoading`。
- 相邻 props 用空行分组：核心数据、状态、回调、样式和透传 props。
- 必须定义 `interface XxxProps`；只有完全透传的组件可用继承型 props。

## 状态与副作用

- UI 组件不允许发请求。
- 非 Modal/Drawer 的业务组件可以通过 hooks 请求，但优先由页面 hook 下发数据。
- 业务 Modal/Drawer 通过 Nice Modal 管理生命周期；初始数据通过 `NiceModal.show()` 参数传入，首次打开后再执行查询。
- 列表选择、表单编辑这类临时状态放组件内部或页面 hook，不放全局 store。
- 事件订阅、定时器、observer 必须在 effect cleanup 中移除。

## 可访问性

- 可点击的非按钮语义使用 `<button type="button">`。
- 弹窗必须处理关闭焦点、Esc 和遮罩行为的预期。
- 图片提供有效 `alt`；装饰性图片使用 `alt=""`。
- 表单控件与 label/校验消息关联。

## 组合示例

```tsx
// src/pages/ProductList/index.tsx
import { Button } from 'antd';
import { NiceModal } from '@ebay/nice-modal-react';
import ProductsSelectModal from '@/components/business/ProductsSelectModal';

export default function ProductList() {
  const handleOpenSelect = () => {
    NiceModal.show(ProductsSelectModal, {
      onSuccess: handleSelect,
    });
  };

  return <Button onClick={handleOpenSelect}>选择商品</Button>;
}
```

页面不维护弹窗开关，也不感知弹窗内部请求细节；business 组件不感知来源页面的私有状态。

## Less 作用域

```less
// src/components/ui/DescriptionDrawer/index.less
.description-drawer {
  .header {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }

  .body {
    padding: var(--space-3) 0;
  }

  .footer {
    display: flex;
    justify-content: flex-end;
  }
}
```

规则：

- 不使用 CSS Modules，不写 `import styles from './index.less'`。
- 不使用 BEM，不写 `.description-drawer__header` 或 `.description-drawer--active`。
- 组件样式根类名必须全局唯一，并挂到组件真实根节点或 Ant Design 的 `className` / `rootClassName` 上。
- 所有组件选择器都嵌套在唯一根类名内部；禁止在组件 Less 中写裸元素、裸全局类或顶层 `.header`。
- 组件内部语义类名可使用 `.header`、`.body`、`.footer`、`.item`；因为 Less 会编译成 `.description-drawer .header`。
- 状态类使用 `is-active`、`is-selected`、`has-error`；动态 className 必须通过 `classnames` 生成。
- 外部如需覆盖，应通过组件 props、CSS 变量或约定的根类名；不要跨组件选择其内部节点。
