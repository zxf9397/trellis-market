---
name: shared-code-quality
description: 可读性、复杂度、注释和错误处理
paths:
  - frontend/src/
---

# 代码质量

## 函数与组件

- 一个组件只解决一件事；超过 250 行时优先拆分局部组件或 hook。
- 一个函数只处理一层抽象。
- 嵌套不超过 3 层；使用 early return。
- 不写只有调用处不同的近似重复逻辑。
- 优先声明式数据映射，不写重复 if/else 字典。
- 不硬编码状态、类型码、来源、级别和业务字典；使用 `as const` 常量对象收口。

## 注释

API 类型、接口函数和字段必须写 JSDoc；字段名、类型和注释以接口文档为准。复杂业务逻辑必须写简洁但可理解的注释，解释业务规则、状态流转或为什么选择当前实现。普通实现注释解释为什么，不重复代码在做什么：

```ts
// 并发请求会同时收到 401；这里用单飞刷新避免多次跳转。
```

不提交：

- 注释掉的死代码。
- TODO 没有负责人、任务链接或移除条件。
- 历史变更叙事。
- 与业务无关的 `console.log` 调试日志。

## 错误处理

- 可预期错误给用户可行动提示。
- 不可预期错误上报并显示通用错误。
- 不吞异常；即使 UI 需要静默，也记录降级原因。
- 不把原始技术错误直接展示给用户。
- 异步流程必须有 loading、error、empty、success 四种状态。

## 命名

- boolean：`isLoading`、`hasPermission`、`canSubmit`。
- 事件 props：`onChange`、`onClose`。
- handler：`handleSubmit`、`handleSelect`。
- API：`getUser`、`listProducts`。
- 避免无意义词：`data2`、`temp`、`handleClick2`。
