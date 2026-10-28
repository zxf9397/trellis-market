---
name: frontend-testing-quality
description: 测试触发规则、格式化、pre-commit、性能、安全和验收检查
paths:
  - frontend/src/components/ui/
  - frontend/src/components/business/
  - frontend/src/hooks/
  - frontend/src/utils/
  - frontend/src/api/
  - frontend/package.json
  - frontend/.husky/
  - frontend/.prettierrc.mjs
  - frontend/.prettierignore
---

# 质量门禁

## 基础检查与验证边界

每次代码变更后，默认必须完成基础检查：

```bash
pnpm lint
pnpm typecheck
```

这类检查用于发现语法、导入、类型和静态规则问题；发现错误时应修复后再次执行。

默认不执行：

- `pnpm test`
- `pnpm test:watch`
- `pnpm coverage`
- `pnpm build`

默认也不新建 `*.test.ts`、`*.test.tsx`、`*.spec.ts`、`*.spec.tsx`，不运行 `vitest`。

测试和构建由开发人员决定。只有开发人员明确要求时才生成或执行测试，例如：

- “帮我测试一下。”
- “测试验证一下。”
- “跑一下测试。”
- “补充单元测试。”
- “覆盖这个场景的测试。”

只有开发人员明确要求时才执行构建，例如：

- “帮我 build 验证一下。”
- “执行构建检查。”
- “打包验证。”

用户要求后，按变更范围选择 targeted test 或全量测试；仍然不要把无关测试文件一次性补齐。

基础检查通过后，完成说明中报告结果，并可以询问：“是否需要继续执行 test / build 验证？”

## 可用命令

`package.json` 至少提供：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint .",
    "format": "prettier . --write",
    "format:check": "prettier . --check",
    "typecheck": "tsc -b --pretty",
    "test": "vitest run",
    "test:watch": "vitest",
    "coverage": "vitest run --coverage",
    "prepare": "husky"
  }
}
```

默认执行基础检查：

```bash
pnpm lint
pnpm typecheck
```

开发人员明确要求后，再按变更范围执行扩展验证：

- 全量：`pnpm lint && pnpm typecheck && pnpm test && pnpm build`
- 纯样式：`pnpm lint`，可选 `pnpm build`
- API/store/router：`pnpm lint && pnpm typecheck`，可选 `pnpm test && pnpm build`
- 配置文件：`pnpm lint && pnpm typecheck`，可选 `pnpm build` 和本地 preview 验证

未要求时，不自动执行测试或构建；也不要用 pre-commit 隐式触发测试。

## Prettier

`.prettierrc.mjs` 作为唯一格式来源：

```js
const config = {
  semi: true,
  singleQuote: true,
  trailingComma: 'all',
  printWidth: 100,
  tabWidth: 2,
  endOfLine: 'lf',
};

export default config;
```

`.prettierignore` 至少包含：

```text
node_modules
dist
coverage
pnpm-lock.yaml
package-lock.json
*.local
```

规则：

- 代码格式争议以 Prettier 输出为准。
- 不在 ESLint 里重复可自动修复的格式规则。
- 新增生成文件加入 ignore。

## Husky + lint-staged

`.husky/pre-commit` 只做快速门禁：

```sh
pnpm exec lint-staged
```

`package.json` 推荐：

```json
{
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix"
    ],
    "*.{js,mjs,cjs,less,css,json,md}": [
      "prettier --write"
    ]
  }
}
```

规则：

- pre-commit 只做 lint 和格式化，不运行 Vitest，不跑完整 build。
- 修复后的文件必须重新 stage。
- 提交信息使用 Conventional Commits，描述必须使用中文：`feat(product-list): 新增商品关键字搜索`。

## 测试策略

以下策略仅在开发人员明确要求检查、测试或补充测试时执行。

优先测试：

- `src/utils/` 纯函数。
- `src/hooks/` 数据编排和状态转换。
- ahooks `useRequest` 的自动查询、manual 动作、`run` 内 async service 成功分支、refresh、loading 和局部错误态。
- `src/components/ui/` 的 props、事件、可访问性。
- `src/components/business/` 的关键交互。
- Nice Modal 业务弹窗/抽屉的打开、确认、成功、统一错误后保留弹窗、loading 锁定和关闭清理。
- API 错误码、401、上传失败、分页合并。
- ProTable 列表分页、搜索参数转换、空参数过滤、竞态取消和失败空态。
- Ant Form 校验、初始化、联动、提交 loading、失败保留输入和成功重置。
- typed client 的响应解包和参数透传。
- `AbortController` 取消请求后不弹错误、不影响新请求。
- 路由守卫的登录/权限分支。

UI 测试使用 Vitest + Testing Library：

- 测试用户可见行为，不测试实现细节。
- 查询优先 `getByRole`、`getByLabelText`，最后才 `data-testid`。
- mock HTTP 边界，不 mock `ahooks useRequest` 本身。
- 不测试组件内部是否维护 loading/data state；测试用户可见的 loading、empty、error、success 行为。
- 异步断言使用 `findBy` 或 `waitFor`。
- 每个测试后清理定时器、observer、订阅、store 和 Nice Modal 状态。

## 样式检查

- [ ] 组件、页面和布局的 `index.less` 都用唯一根类名包裹。
- [ ] 根类名已挂到真实根节点或 Ant Design 的 portal 根节点。
- [ ] 没有使用 CSS Modules 和 `styles.xxx`。
- [ ] 没有 BEM 类名。
- [ ] 响应式断点使用 `mixins.less` 中的 `.media-down()` / `.media-up()`，并优先保持 PC 优先。
- [ ] 动态 className 使用 `classnames`。
- [ ] 没有裸顶层业务选择器或全局元素选择器。
- [ ] Less 已通过对应入口直接导入。

## 性能

- 路由页面默认懒加载。
- 列表项避免创建不稳定 inline object/function 传给 memo 子组件；先测量，不提前优化。
- 大列表使用虚拟滚动。
- 图片设置 width/height、lazy loading 和合适格式。
- 避免把大依赖塞进首屏；优先动态 import。
- 只在真实性能数据证明必要时引入 memo/useMemo/useCallback。

## 安全

- 不把 secret 放前端 env。
- 渲染用户输入交给 React escaping；必须富文本时使用白名单 sanitizer。
- token 存储策略必须维护在 `.trellis/spec/` 对应前端规范中。
- 所有外部跳转使用固定协议白名单。
- 依赖新增必须 review license、维护状态和传递体积。
- 错误提示不暴露堆栈、SQL、内部地址或 requestId 以外的调试细节。
