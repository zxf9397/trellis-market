---
name: frontend-environment-config
description: Vite env、运行时配置、Zod 校验和 feature flags
paths:
  - frontend/.env
  - frontend/.env.development
  - frontend/.env.production
  - frontend/.env.example
  - frontend/vite.config.ts
  - frontend/public/config.js
  - frontend/src/config/
  - frontend/src/types/env.d.ts
---

# 环境与配置

## 文件职责

| 文件 | 是否提交 | 用途 |
| --- | --- | --- |
| `.env` | 可以提交 | 所有环境共享的非敏感默认值 |
| `.env.development` | 可以提交 | `vite` 本地开发默认值 |
| `.env.production` | 可以提交 | 生产构建的非敏感默认值 |
| `.env.example` | 必须提交 | 完整变量清单、类型、示例和说明 |
| `.env.local` / `.env.development.local` | 不提交 | 个人本地覆盖或临时联调地址 |
| `public/config.js` | 提交模板 | 可选部署期注入的非敏感运行时配置 |

## 命名

- 浏览器可见变量必须使用 `VITE_` 前缀。
- 使用大写 `SCREAMING_SNAKE_CASE`。
- URL 变量以 `_URL` 结尾，开关使用 `VITE_FEATURE_*`。
- 禁止把 secret、私钥、服务端签名密钥或内部管理 token 放进任何 `VITE_*` 或 `public/config.js`。

`.env.example` 示例：

```bash
# 应用
VITE_APP_NAME="Example"
VITE_API_BASE_URL="https://api.example.com"

# 功能开关
VITE_FEATURE_NEW_PRODUCT_LIST="false"
```

`public/config.js` 单独维护：

```js
window.__APP_CONFIG__ = {
  apiBaseUrl: "https://api.example.com",
};
```

## 编译期配置收口

所有 `import.meta.env` 只允许出现在 `src/config/env.ts`。其他代码 import 解析后的配置对象。

```ts
// src/config/env.ts
import { z } from 'zod';

const envSchema = z.object({
  VITE_APP_NAME: z.string().min(1),
  VITE_API_BASE_URL: z.string().url(),
  VITE_FEATURE_NEW_PRODUCT_LIST: z
    .enum(['true', 'false'])
    .default('false')
    .transform((value) => value === 'true'),
});

const parsed = envSchema.safeParse(import.meta.env);

if (!parsed.success) {
  const issues = parsed.error.issues
    .map((issue) => `${issue.path.join('.')}: ${issue.message}`)
    .join('\n');

  throw new Error(`Invalid environment variables:\n${issues}`);
}

export const env = {
  appName: parsed.data.VITE_APP_NAME,
  apiBaseUrl: parsed.data.VITE_API_BASE_URL.replace(/\/$/, ''),
  featureNewProductList: parsed.data.VITE_FEATURE_NEW_PRODUCT_LIST,
} as const;

export type Env = typeof env;
```

规则：

- 应用启动时立即校验，不允许请求失败后才发现配置错误。
- 校验后输出窄化类型，禁止到处做 `Boolean(import.meta.env.X)`。
- 除 `window.__APP_CONFIG__` 外，`env.ts` 不读取 DOM、localStorage 或发起请求。
- 需要按构建目标变化的行为放 feature flags，不在页面里写 `import.meta.env.MODE === 'production'`。

## 一镜像多环境

如果同一 Docker image 要部署到多环境，编译期变量只保留不随环境变化的值；环境相关 API 地址放到 `public/config.js`。

```html
<!-- index.html -->
<script src="/config.js"></script>
```

```ts
// src/config/env.ts
import { z } from 'zod';

type RuntimeConfig = Partial<{
  apiBaseUrl: string;
  featureNewProductList: boolean;
}>;

declare global {
  interface Window {
    __APP_CONFIG__?: RuntimeConfig;
  }
}

const envSchema = z.object({
  VITE_APP_NAME: z.string().min(1),
  VITE_API_BASE_URL: z.string().url(),
  VITE_FEATURE_NEW_PRODUCT_LIST: z
    .enum(['true', 'false'])
    .default('false')
    .transform((value) => value === 'true'),
});

const parsed = envSchema.safeParse({
  ...import.meta.env,
  // 运行时配置优先生效。
  VITE_API_BASE_URL: window.__APP_CONFIG__?.apiBaseUrl ?? import.meta.env.VITE_API_BASE_URL,
  VITE_FEATURE_NEW_PRODUCT_LIST:
    window.__APP_CONFIG__?.featureNewProductList?.toString() ??
    import.meta.env.VITE_FEATURE_NEW_PRODUCT_LIST,
});

if (!parsed.success) {
  const issues = parsed.error.issues
    .map((issue) => `${issue.path.join('.')}: ${issue.message}`)
    .join('\n');

  throw new Error(`Invalid environment variables:\n${issues}`);
}

export const env = {
  appName: parsed.data.VITE_APP_NAME,
  apiBaseUrl: parsed.data.VITE_API_BASE_URL.replace(/\/$/, ''),
  featureNewProductList: parsed.data.VITE_FEATURE_NEW_PRODUCT_LIST,
} as const;
```

规则：

- `public/config.js` 只包含非敏感配置，且必须在部署流水线中可替换。
- 运行时配置必须进入同一个 Zod schema 校验。
- `env.ts` 负责合并优先级：运行时配置 > mode env > 共享 env > 默认值。
- `window.__APP_CONFIG__` 是 `env.ts` 中唯一允许读取的运行时全局配置；localStorage 和 DOM 仍然禁止读取。

## Vite 构建相关

- `envPrefix` 必须保持 `VITE_`，不要扩大到任意前缀。
- API 代理只用于本地开发，不在业务代码里假设代理路径。
- import alias、server port、proxy、build sourcemap 应集中在 `vite.config.ts`。
- 新增构建插件必须在 README 说明用途和影响。
