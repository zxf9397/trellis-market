# Trellis Marketplace

可复用的 Trellis spec 模板仓库。

## Structure

```text
index.json
specs/
└── react-antd-frontend/
    ├── README.md
    ├── frontend/
    ├── guides/
    └── shared/
```

`index.json` 中的 `path` 相对仓库根目录。当前模板路径是：

```text
specs/react-antd-frontend
```

## Usage

```bash
trellis init --registry gh:zxf9397/trellis-market#main --template react-antd-frontend
```

已有项目只补齐不存在的 spec 文件：

```bash
trellis init --registry gh:zxf9397/trellis-market#main --template react-antd-frontend --append
```

### Direct mode

```bash
trellis init --registry gh:zxf9397/trellis-market/specs/react-antd-frontend#main
```
