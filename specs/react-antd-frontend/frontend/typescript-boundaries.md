---
name: frontend-typescript-boundaries
description: TypeScript strict、类型组织、ambient 声明和 ESLint 边界
paths:
  - frontend/src/types/
  - frontend/src/constants/
  - frontend/src/utils/
  - frontend/tsconfig.json
  - frontend/tsconfig.app.json
  - frontend/tsconfig.node.json
  - frontend/eslint.config.mjs
---

# TypeScript 与模块边界

## TypeScript 配置

`tsconfig.json` 只做 references，不混装浏览器和 Node 配置：

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ]
}
```

应用代码要求：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "resolveJsonModule": true
  }
}
```

规则：

- 不关闭 `strict`。
- 不用 `any` 兜底；unknown 数据必须先窄化。
- 禁止运行时代码使用 `void` 操作符。
- 具体函数声明、方法或函数表达式的无返回值返回注解必须省略，不要写 `: void`；`void` 只保留在类型契约中，例如 `onClose?: () => void`。
- 故意不等待的 Promise 直接调用 `requestPage()`；不要写 `void requestPage()`。需要处理 rejection 时使用 `await try/catch` 或 `.catch()`。
- 公共导出函数如果有返回值，写显式返回类型；无返回值时省略 `: void`，让 TypeScript 推断。
- 联合状态使用 discriminated union，不用多个可空字段表达互斥状态。
- 枚举优先使用 `as const` 对象或字符串 union，除非需要运行时双向映射。
- 组件 props 用 `interface`；复杂工具类型放 `src/types/`。

## Void 返回类型

无返回值的具体函数不要显式写 `: void`：

```ts
// ok
export function useDocumentTitle(title?: string) {
  useEffect(() => {
    if (!title) {
      return;
    }

    const previousTitle = document.title;
    document.title = title;

    return () => {
      document.title = previousTitle;
    };
  }, [title]);
}
```

```ts
// bad
export function useDocumentTitle(title?: string): void {
  // ...
}
```

这条规则只约束具体函数实现。类型契约中的回调返回类型仍保留 `void`：

```ts
// ok
interface ExampleProps {
  onCancel?: () => void;
}
```

有返回值的公共导出函数仍写显式返回类型：

```ts
export async function getUser(id: string): Promise<User> {
  return client.get<User>(`/users/${id}`);
}
```

## 类型文件

```text
src/types/
├── api.ts       # ApiResponse<T>、PageResult<T>
├── env.d.ts     # ImportMetaEnv
├── global.d.ts  # window.__APP_CONFIG__ 等 ambient
└── index.ts
```

规则：

- `*.d.ts` 只声明类型，不包含运行时代码。
- `env.d.ts` 只声明 `ImportMetaEnv`，不替代 `src/config/env.ts`。
- `global.d.ts` 尽量少用；新增全局对象必须写明确类型。
- API 类型靠近领域接口；只有跨领域类型放 `src/types/`。
- 不从组件文件 import 页面或 API 类型造成循环引用；必要时建立 `types.ts`。

## 常量与工具

- 禁止 `enum` 和 `const enum`；业务枚举使用 `as const` 对象，详见 [constants-enums.md](./constants-enums.md)。
- 枚举常量对象名使用 PascalCase，Key 使用 PascalCase，Value 必须与后端传输值一致。
- 运行时引用枚举成员，不硬编码状态、类型码和字典值。
- `constants/enums.ts` 只放跨模块业务字典。
- `constants/http.ts` 放响应码、HTTP status、通用错误 key。
- `constants/index.ts` 只做 re-export。
- `utils/` 保持纯函数：相同输入得到相同输出，不依赖组件、store、路由和 API。
- `storage.ts` 收口 localStorage/sessionStorage key、JSON parse、过期和异常。
- `validator.ts` 使用 Zod schema 或类型守卫，不返回 `any`。
- `format.ts` 处理展示格式，不做请求、缓存或业务状态修改。

## ESLint Flat Config 边界

`eslint.config.mjs` 至少包含：

- TypeScript recommended + React hooks + React refresh。
- Prettier compatibility。
- 禁用 `enum`：使用 `no-restricted-syntax` 匹配 `TSEnumDeclaration`。
- 禁用 `void` 操作符：启用 `no-void`，不开启 `allowAsStatement`。
- import order。
- boundaries。

推荐边界：

```js
// 简化示例
import boundaries from 'eslint-plugin-boundaries';

export default [
  // ...
  {
    files: ['src/**/*.{ts,tsx}'],
    plugins: { boundaries },
    settings: {
      'boundaries/elements': [
        { type: 'ui', pattern: 'src/components/ui/**' },
        { type: 'business', pattern: 'src/components/business/**' },
        { type: 'api', pattern: 'src/api/**' },
        { type: 'store', pattern: 'src/store/**' },
        { type: 'hooks', pattern: 'src/hooks/**' },
        { type: 'pages', pattern: 'src/pages/**' },
        { type: 'router', pattern: 'src/router/**' },
      ],
    },
    rules: {
      'boundaries/element-types': ['error', {
        default: 'disallow',
        rules: [
          { from: ['ui'], allow: ['ui'] },
          { from: ['business'], allow: ['ui', 'api', 'store', 'hooks'] },
          { from: ['hooks'], allow: ['api', 'store'] },
          { from: ['pages'], allow: ['ui', 'business', 'hooks', 'store', 'api', 'router'] },
          { from: ['router'], allow: ['pages', 'store', 'router'] },
        ],
      }],
    },
  },
];
```

必须用 lint 阻止：

- 页面互相引用私有实现。
- UI 组件引用 API/store。
- 循环 import。
- 直接引用深层路径绕过目录入口。
- 关闭规则的 `eslint-disable`，除非带原因和到期说明。
- 运行时代码中的 `void requestPage()`。

## Import 顺序

推荐顺序：

1. Node 或外部包
2. alias 类型
3. alias 运行时模块
4. 相对模块
5. 样式文件

同组按字母排序，类型导入优先使用 `import type`。
