# React + Vite Frontend Spec Template

这是一个 Trellis spec template，目标技术栈：

- Vite + React 18 + TypeScript
- React Router
- Ant Design Form 表单
- Ant Design + Pro Components 表格/列表
- 全局 Less + 唯一根类名隔离
- classnames 动态 className
- Axios API 层
- ahooks `useRequest` 数据获取
- Zustand 风格的全局状态
- @ebay/nice-modal-react 业务弹窗/抽屉封装
- `as const` 业务枚举与常量对象
- dayjs 时间格式化
- Zod 环境变量校验
- ESLint Flat Config + Prettier + Husky + lint-staged

## 安装

```bash
# marketplace 模式
trellis init --registry gh:your-org/your-spec-repo/marketplace --template react-antd-frontend

# 已有项目只补齐不存在的 spec 文件
trellis init --registry gh:your-org/your-spec-repo/marketplace --template react-antd-frontend --append
```

如果把本目录作为 direct source 使用，则不需要 `marketplace/index.json`：

```bash
trellis init --registry gh:your-org/your-spec-repo/marketplace/specs/react-antd-frontend
```

## 项目结构约定

模板默认约束如下仓库形态：

```text
frontend/
├── README.md
├── .env
├── .env.development
├── .env.production
├── .env.example
├── .prettierrc.mjs
├── .prettierignore
├── eslint.config.mjs
├── vite.config.ts
├── tsconfig.json / tsconfig.app.json / tsconfig.node.json
├── package.json
├── .husky/pre-commit
├── public/config.js
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── api/
    ├── assets/
    ├── components/{ui,business}/
    ├── config/
    ├── constants/
    ├── hooks/
    ├── layouts/
    ├── mocks/
    ├── pages/
    ├── router/
    ├── store/
    ├── styles/
    ├── types/
    └── utils/
```

安装后按项目调整：

1. 把 `frontend/` 路径改为真实前端包路径；如果是单包根目录，可把 `frontend/` 前缀删掉。
2. 补充真实的响应包装、错误码、登录接口和权限模型。
3. 删除不使用的规则，例如没有 MSW 时删除 mocks 相关内容。
4. 维护 `frontend/README.md`，让它只描述项目入口信息；实现规范统一放在 `.trellis/spec/`。

## README 与 Trellis Spec 的边界

使用 Trellis 后，不要在 `src/README.md` 重复维护编码规范。

| 位置 | 用途 |
| --- | --- |
| `frontend/README.md` | 人类项目入口文档：项目简介、启动方式、脚本、环境变量、构建、部署入口、常见问题 |
| `.trellis/spec/**` | 唯一编码规范来源：分层规则、依赖方向、目录职责、命名约定、组件/API/状态/测试约束 |

如果仓库只有一个前端包，`frontend/README.md` 也可以放在仓库根目录；如果是 monorepo，则放在 `frontend/` 这种包根目录下。
