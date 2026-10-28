---
name: shared-git-review
description: 分支、提交、PR 和 review 要求
---

# Git 与 Review

## 提交

- 使用 Conventional Commits：`feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `chore`。
- scope 使用页面、领域或模块，保留英文或常见缩写：`product-list`、`api`、`auth`、`router`。
- 描述必须使用中文，结尾不加句号：`feat(product-list): 新增商品关键字搜索`。
- 修改原因、影响范围或 breaking change 写在 body 中，body 也使用中文。
- 一个提交只表达一个意图。
- 不提交 secret、本地产物、临时 debug 日志。
- 大文件和生成文件先确认是否应入库。

## PR

- PR 描述包含目的、变更点、验证方式、截图/录屏和风险。
- UI 变更附 before/after。
- API/权限变更列出接口、错误码、兼容性；无接口文档时说明 Mock 范围和阻塞项。
- 配置变更列出环境影响和回滚方式。

## 提交示例

```text
feat(product-list): 新增商品关键字搜索
```

```text
fix(auth): 修复并发 401 时重复跳转登录页

并发请求同时收到 401 时，增加单飞刷新和跳转防抖，
避免触发多次登录态清理和重复导航。
```

```text
refactor(api): 统一订单列表分页参数

将 page 转换为后端协议使用的 pageNo，并过滤空筛选条件。
```

## Review 检查

- 是否符合目录边界和依赖方向？
- 是否直接读取 `import.meta.env` 或直接使用 axios？HTTP 是否都走类型增强 client？
- 是否有 loading/error/empty 状态？
- 是否处理 401 和业务错误码？
- 是否有测试和类型验证？
- 提交类型、scope 和中文描述是否符合 Conventional Commits？
- 是否引入新的全局状态、全局样式或不必要依赖？
