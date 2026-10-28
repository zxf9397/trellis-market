---
name: frontend-modal-drawer
description: Nice Modal 封装业务弹窗、抽屉和异步确认流
paths:
  - frontend/src/components/business/
  - frontend/src/main.tsx
  - frontend/src/pages/
  - frontend/package.json
---

# 业务弹窗与抽屉

## 强制约定

弹窗、抽屉、确认框等**业务组件**必须使用 `@ebay/nice-modal-react` 封装：

- 依赖放 `dependencies`。
- `NiceModal.Provider` 只在 `src/main.tsx` 挂载一次。
- 组件用 `NiceModal.create()` 创建，默认导出封装后的组件。
- 组件内部通过 `NiceModal.useModal()` 获取 `visible`、`show`、`hide`、`resolve`、`remove`。
- 调用方用 `NiceModal.show()` 打开，不在页面里维护 `open` state。
- 业务弹窗 props 不接受 `open`、`visible`、`onOpenChange`、`onCancel` 这类展示开关。
- 纯展示的 UI 抽屉/弹窗仍可作为 `src/components/ui/` 的受控组件；但只要承载业务请求、确认动作、登录、下单、审批等业务流，必须按本规范使用 Nice Modal。

安装：

```bash
pnpm add @ebay/nice-modal-react
```

`main.tsx`：

```tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { NiceModal } from '@ebay/nice-modal-react';
import { StyleProvider } from '@ant-design/cssinjs';
import { ConfigProvider } from 'antd';
import { RouterProvider } from 'react-router-dom';
import { router } from '@/router';
import { lightTheme } from '@/styles/antd/antd';
import '@/styles/global.less';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <ConfigProvider theme={lightTheme}>
      <StyleProvider hashPriority="low">
        <NiceModal.Provider>
          <RouterProvider router={router} />
        </NiceModal.Provider>
      </StyleProvider>
    </ConfigProvider>
  </StrictMode>,
);
```

Provider 顺序原则：

- `NiceModal.Provider` 必须在 `ConfigProvider` / `StyleProvider` 内部，保证业务弹窗能拿到主题和样式上下文。
- 只允许在 `main.tsx` 挂载一次；页面、组件、store 不允许再包 `NiceModal.Provider`。

## 目录与命名

```text
src/components/business/AfterSaleCancelModal/
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

- 目录名使用 PascalCase，并以 `Modal`、`Drawer`、`ConfirmDialog` 结尾。
- Less 根类名使用对应 kebab-case，例如 `AfterSaleCancelModal` 使用 `.after-sale-cancel-modal`。
- 所有样式都嵌套在该唯一根类名内，并挂到 Ant Design 的 `className` 或 `rootClassName`。
- 不使用 BEM，不使用 CSS Modules。
- 动态 className 统一使用 `classnames`，例如 `classNames('after-sale-cancel-modal', { 'is-loading': loading })`。

## 标准异步确认弹窗

```tsx
import { NiceModal } from '@ebay/nice-modal-react';
import { Modal, message } from 'antd';
import { cancelRefund } from '@/api/afterSale';
import { useRequest } from 'ahooks';
import './index.less';

interface AfterSaleCancelModalProps {
  orderProductId: string;
  onSuccess?: VoidFunction;
}

const AfterSaleCancelModal = NiceModal.create(
  ({ orderProductId, onSuccess }: AfterSaleCancelModalProps) => {
    const modal = NiceModal.useModal();
    const { loading, run } = useRequest(
      async () => {
        try {
          await cancelRefund({ order_product_id: orderProductId });

          // 请求成功后才执行业务动作。
          onSuccess?.();
          message.success('已取消本次售后');
          modal.resolve(true);
          modal.hide();
        } catch {
          // 统一响应拦截器已处理错误提示。
        }
      },
      {
        manual: true,
      },
    );

    return (
      <Modal
        title="取消售后"
        open={modal.visible}
        className="after-sale-cancel-modal"
        cancelButtonProps={{ disabled: loading }}
        okButtonProps={{ loading }}
        onOk={run}
        onCancel={modal.hide}
        afterClose={modal.remove}
        maskClosable={!loading}
        closable={!loading}
        keyboard={!loading}
        centered
      >
        是否确认取消本次售后申请？
      </Modal>
    );
  },
);

export default AfterSaleCancelModal;
```

异步动作必须满足：

- 异步确认动作使用 ahooks 的 `useRequest`，并设置 `manual: true`；成功后的业务动作写在 async service 内的 `await request(...)` 之后，确认时调用 `run()`。
- loading 期间禁止关闭：禁用取消按钮、禁止 mask 关闭、禁止 Esc、隐藏 close icon。
- 成功后按顺序执行业务回调、用户提示、`modal.resolve()`、`modal.hide()`。
- 接口报错由统一响应拦截器处理；失败时不 resolve、不 hide，保留弹窗，且不重复弹错误提示。
- `afterClose` 使用 `modal.remove`，确保卸载并清理 Nice Modal 状态。
- 弹窗内部不直接改页面 state、路由或全局 store；通过 props 回调通知调用方。
- API 调用必须来自 `src/api/`，不得在弹窗内直接使用 Axios。

## 调用方式

```tsx
import { Button } from 'antd';
import { NiceModal } from '@ebay/nice-modal-react';
import AfterSaleCancelModal from '@/components/business/AfterSaleCancelModal';

export function CancelButton({ orderProductId }: { orderProductId: string }) {
  const handleCancelAfterSale = () => {
    NiceModal.show(AfterSaleCancelModal, {
      orderProductId,
      onSuccess: () => {
        refreshOrderDetail();
      },
    });
  };

  return <Button onClick={handleCancelAfterSale}>取消售后</Button>;
}
```

如果调用方需要等待结果，必须捕获取消/失败分支：

```tsx
const confirmed = await NiceModal.show(ConfirmDeleteModal, { id });

if (confirmed) {
  await refresh();
}
```

规则：

- 不要求显式添加 `void`。`NiceModal.show(...)` 可以直接调用；只有调用方需要根据弹窗结果分支时才使用 `await`。
- 需要根据结果分支时 `await NiceModal.show(...)`。
- `NiceModal.show()` 的 args 只传业务参数和结果回调，不传 `open`。
- 同一个业务操作在多个页面复用时，弹窗放 `src/components/business/`。
- 只属于一个页面的业务弹窗仍放该页面目录，但封装方式不变。

## Drawer 模式

```tsx
const ProductDetailDrawer = NiceModal.create(({ productId }: ProductDetailDrawerProps) => {
  const modal = NiceModal.useModal();

  return (
    <Drawer
      title="商品详情"
      open={modal.visible}
      className="product-detail-drawer"
      onClose={modal.hide}
      afterOpenChange={(open) => {
        if (!open) {
          modal.remove();
        }
      }}
      destroyOnClose
      width={720}
    >
      <ProductDetailContent productId={productId} />
    </Drawer>
  );
});
```

规则：

- Drawer 没有 `afterClose` 时使用 `afterOpenChange(false)` 调用 `modal.remove()`。
- 异步提交中的 Drawer 也必须锁定关闭入口。
- Drawer 内部复杂内容可拆成子组件；业务请求放在领域 hook 或 drawer 内 hook，不放父页面。

## 状态与生命周期

- 弹窗的打开/关闭状态由 Nice Modal 管理，不写入 Zustand。
- 业务结果通过回调或 Promise 返回；不要把“上一个弹窗结果”塞进全局 store。
- 表单初始值优先通过 `NiceModal.show()` 参数传入。
- 弹窗卸载后不允许继续 setState；异步完成前用户不可关闭时必须锁定关闭入口。
- 连续打开同一弹窗时，不依赖上次残留的表单、分页、选择或请求结果。
- 认证弹窗、全局通知弹窗可以由任意远程入口触发，但仍通过 `NiceModal.show()`，不通过 store 里的 `globalModal` 状态驱动。

## 测试要点

- `NiceModal.show()` 后弹窗标题、正文和默认值正确。
- 确认动作触发领域 API。
- 成功后调用 `onSuccess`、提示、resolve 并关闭。
- API 失败时弹窗保持打开并展示错误。
- loading 中取消按钮、mask、Esc、close icon 均不可关闭。
- 关闭后内部表单和请求状态不会影响下一次打开。
- Drawer 关闭后触发 remove/清理。
