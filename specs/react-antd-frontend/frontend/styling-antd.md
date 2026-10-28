---
name: frontend-styling-antd
description: 全局 Less、唯一根类名作用域、design token、主题和 Ant Design 覆写
paths:
  - frontend/src/**/*.less
  - frontend/src/components/
  - frontend/src/pages/
  - frontend/src/layouts/
  - frontend/src/styles/
  - frontend/src/main.tsx
---

# 样式与 Ant Design

## 文件职责

```text
src/styles/
├── antd/
│   ├── antd.ts
│   └── override.less
├── mixins.less
├── tokens.less
├── themes.less
└── global.less
```

| 文件 | 职责 |
| --- | --- |
| `tokens.less` | 颜色、间距、圆角、阴影、字体、z-index token |
| `mixins.less` | 断点 map、媒体查询 mixin、可复用 Less mixins |
| `themes.less` | light/dark 或品牌主题差异 |
| `global.less` | reset、基础排版、滚动条、全局工具类 |
| `antd/antd.ts` | Ant Design `ThemeConfig` 主路径 |
| `antd/override.less` | 无法通过 token 表达的兜底覆写 |

规则：

- 组件和页面样式使用普通全局 Less：`index.less`。
- 每个 `index.less` 必须用组件/页面唯一的根类名包裹，禁止裸顶层业务选择器。
- 页面和组件不直接写 `.ant-...` 覆写。
- 需要 Ant Design 覆写时优先改 `antd.ts`；Less 兜底必须限定作用域。
- 禁止 `!important`，除非记录原因且无法用 selector specificity 解决。

## Less 作用域

组件样式使用普通全局 Less，但必须通过唯一根类名手动限定作用域。

```less
// src/pages/ProductList/index.less
.product-list-page {
  display: flex;
  flex-direction: column;
  gap: var(--space-4);

  .filter {
    display: flex;
    align-items: flex-end;
    gap: var(--space-2);
  }

  .table-card {
    padding: var(--space-3);
  }

  .item.is-selected {
    border-color: var(--color-primary);
  }
}
```

```tsx
// src/pages/ProductList/index.tsx
import classNames from 'classnames';
import { Button } from 'antd';
import type { Product } from '@/api/products/types';
import './index.less';

interface ProductListProps {
  products: Product[];
  selectedId?: string;
  onSelect: (productId: string) => void;
}

export default function ProductList({ products, selectedId, onSelect }: ProductListProps) {
  return (
    <div className="product-list-page">
      <section className="filter">{/* ... */}</section>
      <section className="table-card">
        {products.map((product) => (
          <button
            key={product.id}
            type="button"
            className={classNames('item', {
              'is-selected': product.id === selectedId,
            })}
            onClick={() => onSelect(product.id)}
          >
            {product.name}
          </button>
        ))}
      </section>
    </div>
  );
}
```

规则：

- 禁止 CSS Modules，禁止 `import styles from './index.less'`。
- 组件和页面统一 `import './index.less'`。
- 动态 className 必须使用 `classnames`：`classNames('item', { 'is-selected': isSelected })`。
- 禁止手写条件 className 拼接、数组 `filter/join` 或引入 `clsx`。
- 不使用 BEM；禁止 `__`、`--` 组成的 block/element/modifier 命名。
- Less 文件中只能有一个业务顶层选择器，即唯一的根类名；所有业务选择器都嵌套在它内部。
- 根类名由目录名转换：
  - `DescriptionDrawer` -> `.description-drawer`
  - `AuthModal` -> `.auth-modal`
  - `ProductList` 页面 -> `.product-list-page`
  - `AppLayout` -> `.app-layout`
- 根类名必须挂到组件真实根节点；Modal / Drawer 等传送门组件挂到 Ant Design 提供的 `className` 或 `rootClassName`。
- 组件内部类名使用语义名：`.header`、`.body`、`.footer`、`.item`、`.filter`。由于 Less 会嵌套编译，它们不会污染全局。
- 状态类使用 `is-active`、`is-selected`、`is-disabled`、`has-error`，并通过 `classnames` 挂载。
- 嵌套不超过两层；例如 `.product-list-page .filter` 是一层嵌套。更深层结构优先拆组件或补充语义容器。
- 禁止在组件 Less 中写裸元素、裸全局类、`.root`、`.container` 这类可重复名。
- 组件内部 DOM 结构对外部不可依赖；外部覆写只允许通过 props、CSS variables 或明确的根类名定制。

## Design Token

```less
// tokens.less
:root {
  --color-primary: #1677ff;
  --color-success: #52c41a;
  --color-warning: #faad14;
  --color-danger: #ff4d4f;

  --space-1: 4px;
  --space-2: 8px;
  --space-3: 16px;
  --space-4: 24px;

  --radius-md: 8px;
  --shadow-card: 0 2px 8px rgb(0 0 0 / 8%);

  --z-index-dropdown: 1000;
  --z-index-modal: 1100;
}
```

规则：

- 颜色、间距、圆角、阴影、z-index 优先使用 token。
- 不在组件里复制品牌色。
- dark 主题差异放 `themes.less`，不要在业务组件里判断主题。
- token 名称面向用途，不面向具体值：使用 `--color-danger`，不使用 `--red`。

## Media Query Mixin

Less 没有内置媒体查询函数。响应式断点统一封装在 `src/styles/mixins.less`，业务 Less 禁止手写 `@media (min-width: 768px)`、`@media (max-width: 767px)` 这类裸断点。

```less
// src/styles/mixins.less

// 断点按 Ant Design 语义命名；项目可按设计稿调整值。
@breakpoints: {
  sm: 576px;
  md: 768px;
  lg: 992px;
  xl: 1200px;
  xxl: 1600px;
};

// 1. 向下查询（max-width），默认使用，保持 PC 优先。
.media-down(@breakpoint; @content) when (isdefined(@breakpoints)) {
  @bp-value: map-get(@breakpoints, @breakpoint);

  & when (not (isundefined(@bp-value))) {
    // 减 1px，避免相邻断点区间重叠。
    @media (max-width: (@bp-value - 1px)) {
      @content();
    }
  }
}

// 2. 向上查询（min-width），只用于移动优先或特殊覆盖。
.media-up(@breakpoint; @content) when (isdefined(@breakpoints)) {
  @bp-value: map-get(@breakpoints, @breakpoint);

  & when (not (isundefined(@bp-value))) {
    @media (min-width: @bp-value) {
      @content();
    }
  }
}
```

使用时先导入 mixin，并始终在唯一根类名内部调用：

```less
// src/pages/ProductList/index.less
@import '@/styles/mixins.less';

.product-list-page {
  display: flex;
  flex-direction: column;
  gap: var(--space-3);

  // PC 优先：PC 样式写在正常位置。
  .filter {
    display: flex;
    flex-direction: row;
    align-items: flex-end;
    gap: var(--space-2);
  }

  // 小于 md 时覆盖为纵向排列。
  .media-down(md, {
    .filter {
      flex-direction: column;
      align-items: stretch;
    }
  });
}
```

只在确有移动优先或特殊覆盖时使用向上查询：

```less
.product-list-page {
  .table-card {
    max-height: 320px;

    .media-up(lg, {
      max-height: 480px;
    });
  }
}
```

规则：

- 断点 key 只能使用 `mixins.less` 中定义的 `sm`、`md`、`lg`、`xl`、`xxl`。
- 优先使用 `.media-down()`，保持 PC 优先：PC 样式写在正常位置，低断点差异放进 media query。
- `.media-up()` 只用于移动优先或特殊覆盖，需要说明原因。
- 组件和页面禁止硬编码 `768px`、`@media (min-width: ...)`。
- 媒体查询内样式必须嵌套在唯一根类名内。
- 断点调整只能修改 `mixins.less`，不在业务组件里覆盖断点。

## Ant Design 主题

```ts
// src/styles/antd/antd.ts
import type { ThemeConfig } from 'antd';
import { theme } from 'antd';

export const lightTheme: ThemeConfig = {
  algorithm: theme.defaultAlgorithm,
  token: {
    colorPrimary: '#1677ff',
    borderRadius: 8,
  },
  components: {
    Button: {
      controlHeight: 36,
    },
  },
};

export const darkTheme: ThemeConfig = {
  algorithm: theme.darkAlgorithm,
  token: {
    colorPrimary: '#1677ff',
    borderRadius: 8,
  },
};
```

`main.tsx`：

```tsx
<ConfigProvider theme={currentTheme}>
  <StyleProvider hashPriority="low">
    <NiceModal.Provider>
      <RouterProvider router={router} />
    </NiceModal.Provider>
  </StyleProvider>
</ConfigProvider>
```

规则：

- Ant Design 配置只从 `styles/antd/antd.ts` 读取。
- `ConfigProvider` 通常在 `main.tsx` 或 AppLayout 一处配置。
- 页面不包私有 `ConfigProvider`；局部品牌区可封装成 UI 组件。
- `NiceModal.Provider` 放在 `ConfigProvider` / `StyleProvider` 内部，业务弹窗样式仍用唯一根类名包裹。
- 弹窗动态状态类同样使用 `classnames`，并嵌套在唯一根类名内。
- 升级 Ant Design 后先检查 `ThemeConfig` 是否能替代旧 Less 覆写。
