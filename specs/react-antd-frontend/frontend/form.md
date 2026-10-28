---
name: frontend-form
description: Ant Design Form、Form.Item、类型、校验、提交、联动和弹窗表单
paths:
  - frontend/src/pages/
  - frontend/src/components/
  - frontend/src/api/
  - frontend/src/utils/
---

# 表单规范

## 基本约定

- 表单统一使用 Ant Design `Form` / `Form.Item`。
- 表单实例通过 `Form.useForm<Values>()` 创建，不依赖 `onValuesChange` 手工同步每个输入到 state。
- 表单值类型必须显式定义；复杂表单拆分 Search、Create、Update、Action 等独立类型。
- 提交前必须 `await form.validateFields()` 或使用 `form.validateFields()` 抛出的 values；不要使用 `form.getFieldsValue()` 绕过校验。
- 校验规则集中写在 `Form.Item` 的 `rules`；跨字段、异步、复杂规则可提取 validator 或页面 hook。
- 提交函数统一 `handleXxx`；请求成功后的业务动作写在 async service 内 `await request(...)` 之后。
- 接口报错由统一响应拦截器处理；`catch` 只阻止未处理 rejection、维持局部 UI 状态，不重复弹错误提示。
- loading 期间必须禁止重复提交，Modal / Drawer 同时禁止关闭。

## 类型定义

```ts
/** 创建订单表单值 */
export interface CreateOrderFormValues {
  /** 用户名 */
  name: string;
  /** 订单状态 */
  orderStatus: OrderStatusValue;
}

/** 创建订单请求参数 */
export interface CreateOrderParams {
  /** 用户名 */
  userName: string;
  /** 订单状态 */
  order_status: OrderStatusValue;
}
```

规则：

- 表单值类型放在使用它的组件上方、页面 `constants.ts` 或 API `types.ts`；和后端协议相关的提交类型放 `src/api/**/types.ts`。
- 表单专用字段和接口字段不同名时，必须在提交前显式转换。
- 可选字段必须写 `?`；接口文档确认后补充 JSDoc。
- 不要用 `any` 或 `Record<string, any>` 表示表单值。

## 表单结构

```tsx
import { Button, Form, Input, Select, message } from 'antd';
import { OrderStatus, OrderStatusItems, type OrderStatusValue } from '@/constants/enums';
import type { CreateOrderFormValues } from './types';
import './index.less';

interface CreateOrderFormProps {
  submitting?: boolean;
  onFinish: (values: CreateOrderFormValues) => void;
  onCancel?: () => void;
}

export default function CreateOrderForm({
  submitting = false,
  onFinish,
  onCancel,
}: CreateOrderFormProps) {
  const [form] = Form.useForm<CreateOrderFormValues>();

  return (
    <Form
      form={form}
      layout="vertical"
      name="create-order"
      initialValues={{ orderStatus: OrderStatus.Pending.value }}
      onFinish={onFinish}
      className="create-order-form"
      requiredMark
    >
      <Form.Item<CreateOrderFormValues>
        label="用户名"
        name="name"
        rules={[
          { required: true, whitespace: true, message: '请输入用户名' },
          { max: 32, message: '用户名不能超过 32 个字符' },
        ]}
      >
        <Input allowClear placeholder="请输入用户名" maxLength={32} />
      </Form.Item>

      <Form.Item<CreateOrderFormValues>
        label="订单状态"
        name="orderStatus"
        rules={[{ required: true, message: '请选择订单状态' }]}
      >
        <Select
          placeholder="请选择订单状态"
          options={OrderStatusItems.map((item) => ({
            label: item.label,
            value: item.value,
          }))}
        />
      </Form.Item>

      <div className="form-actions">
        <Button onClick={onCancel}>取消</Button>
        <Button type="primary" htmlType="submit" loading={submitting}>
          提交
        </Button>
      </div>
    </Form>
  );
}
```

`Form.Item` 规则：

- 每个字段必须设置 `name`。
- `label` 必须描述业务含义，不使用“请输入”“字段1”。
- 必填字段使用 `rules={[{ required: true }]}`，不单独手写空值校验。
- `whitespace: true` 只用于文本必填项。
- 受控输入的校验消息、placeholder、`maxLength`、`min`、`max`、`precision` 保持一致。
- `initialValues` 只写真正需要默认展示或提交默认值的字段。
- 不要把 `Form.Item` 嵌套在另一个 `Form.Item` 内表达非字段布局；布局用 `Row`、`Col`、`Space` 或样式容器。
- 自定义控件必须通过 `value` / `onChange` 接入 Form。

## 校验规则

```tsx
<Form.Item<CreateOrderFormValues>
  label="确认密码"
  name="confirmPassword"
  dependencies={['password']}
  rules={[
    { required: true, message: '请再次输入密码' },
    ({ getFieldValue }) => ({
      validator: (_, value) => {
        if (!value || getFieldValue('password') === value) {
          return Promise.resolve();
        }

        return Promise.reject(new Error('两次输入的密码不一致'));
      },
    }),
  ]}
>
  <Input.Password placeholder="请再次输入密码" />
</Form.Item>
```

规则：

- 单字段格式、必填、长度、范围写在 `rules`。
- 跨字段校验使用 `dependencies` + `validator`。
- 服务端异步唯一性校验必须防抖，并在提交时再次校验。
- 错误文案说明用户该怎么做，例如“请输入 11 位手机号”。
- 禁止 `validateStatus` / `help` 手工维护普通表单错误，除非控件结构无法用 `Form.Item` 表达。
- 禁止在提交函数里重复复制一套硬编码校验逻辑。

## 提交流程

```tsx
const [orderForm] = Form.useForm<CreateOrderFormValues>();
const { loading, run } = useRequest(
  async () => {
    const values = await orderForm.validateFields();

    try {
      await createOrder(values);

      message.success('创建成功');
      onSuccess?.();
      orderForm.resetFields();
    } catch {
      // 统一响应拦截器已处理错误提示。
    }
  },
  { manual: true },
);

<Form form={orderForm} onFinish={run}>{/* ... */}</Form>
```

提交流程：

- 点击提交或 `onFinish` 后先校验表单。
- 校验通过后转换表单值，只提交接口需要的字段。
- 服务端分页筛选表单可由 ProTable request 接管；业务创建/编辑表单使用上述 `Form` 流程。
- 成功后按业务决定提示、关闭弹窗、清空表单、刷新列表或跳转。
- 失败时保留用户输入和弹窗状态；统一拦截器已提示错误。
- loading 时禁用提交按钮；Modal / Drawer 禁止 mask、Esc 和 close 关闭。
- 重复点击由按钮 loading、防抖或节流阻止。

## 编辑表单与初始值

```tsx
useEffect(() => {
  if (open && record) {
    form.setFieldsValue({
      name: record.name,
      status: record.status,
      description: record.description,
    });
  }
}, [form, open, record]);
```

规则：

- 新建表单使用 `initialValues`。
- 编辑表单在打开后用 `setFieldsValue` 填充后端数据。
- 不把整个后端响应对象直接塞进 form；只填表单字段。
- `setFieldsValue` 前确保 Modal / Drawer 已经挂载。
- `destroyOnClose` / `destroyOnHidden` 或 `resetFields()` 避免上一次编辑残留。
- 服务端返回的日期字符串必须转换成组件需要的类型，例如 `dayjs`。
- 提交前把前端日期对象、range、select 标签等转换成接口协议字段。

## 表单联动

- 简单显示联动使用 `Form.Item noStyle` + `shouldUpdate` render props。
- 跨字段校验使用 `dependencies` + `validator`。
- 字段清空、启用、禁用、选项变化集中写在明确 effect 或 helper。
- 远程 options 已加载后缓存或按需要刷新，不要每次输入都请求。

## ProTable 搜索表单

服务端分页列表的搜索表单由 ProTable 管理，不重复包一层 `Form`。搜索字段、valueType、日期范围转换、空参数过滤和折叠配置见 [table-list.md](./table-list.md)。

## 样式与布局

- 页面表单根类名使用 `<form-name>-form`，并保持全局唯一。
- Modal / Drawer 内表单可以直接使用父级唯一根类名，不额外包表单根类。
- 常规页面表单优先 `layout="vertical"`；字段名较长的后台表单可用 `layout="horizontal"`，并明确 `labelCol` / `wrapperCol`。
- 操作按钮放 `.form-actions` 或明确容器，不在 `Form.Item` 内塞非字段按钮。
- 响应式布局使用 `mixins.less`，不写裸断点。
- 校验错误展示依赖 Form 自带结构。

## Nice Modal 表单

弹窗内表单优先拆成 `components/XxxForm.tsx`。初始值通过 `NiceModal.show()` 传入，提交动作写在 Modal 的 async service 内，loading 来自 `useRequest`。成功后执行 `onSuccess?.()`、`modal.resolve(true)`、`modal.hide()`；loading 中禁止关闭弹窗。API 失败保留弹窗和用户输入，由统一拦截器处理错误提示。

## 可访问性与测试

- 每个 `Form.Item` 必须有 `label`；placeholder 不能替代 label。
- 必填、错误、帮助文案可被读屏获取，不只用颜色区分。
- 破坏性按钮使用 `danger`；提交按钮使用业务动词。

测试必须覆盖：

- 必填、空格、超长、非法格式展示校验信息。
- 合法提交发送正确字段和类型。
- 编辑初始化正确，提交不包含多余字段。
- 联动字段显示、必填、清空逻辑正确。
- loading 中无法重复提交和关闭弹窗。
- 请求失败保留表单值并展示统一错误。
- 成功后关闭、重置、刷新逻辑符合预期。
