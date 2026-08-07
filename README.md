# tiny-teras

Tera 代码生成引擎资产仓（模板 + 模块配置 + registry），供多方共享：

| 消费方 | 用途 |
|---|---|
| `tiny` 主仓 | 生成 core Rust 代码 / migrations / proto |
| `tiny-client-gpui` | 生成 GPUI UI/model 代码 |
| `tiny-client-flutter` | 生成 Flutter UI/model 代码 |

## 快速开始

```bash
# 在各消费仓的同级目录 clone
cd D:/tiny-project
git clone <this-repo> tiny-teras
```

## 目录结构

```
tiny-teras/
├── templates/            # Tera 模板 (core_rust_* / core_flutter_* / core_proto / core_sql_migration / core_workspace_module)
├── modules/              # 模块任务配置 (um / workspace / examples)
├── registry.yaml         # 任务组注册 (backend/frontend/all/core/proto/migrations/workspace_*)
├── generated/            # 生成产物 (gitignored, 运行时产生)
├── backups/              # apply 备份 (gitignored, 运行时产生)
└── CHANGELOG.md
```

## 消费方式（tinyd）

dev_cli (`tinyd`) 通过两个环境变量定位 teras 与目标仓库：

```bash
# teras 资产位置
export TINY_TERAS_DIR=/d/tiny-project/tiny-teras

# 目标仓库根（teras 独立后不再是 teras 的父目录）
export TINY_PROJECT_ROOT=/d/tiny-project/tiny

# 生成
tinyd tera gen --task um

# 应用到目标仓库
tinyd tera apply --task um
```

> 不设置 `TINY_PROJECT_ROOT` 时，回退到 `TINY_TERAS_DIR` 的父目录（兼容旧的"teras 在项目内"布局）。

## 修改模板

1. 编辑 `templates/**/*.tera` 或 `modules/**/*.yaml`
2. 提交 PR 到本仓，各消费方拉取后生效
3. 模板与目标代码结构强相关——修改模板需同步修改消费方的代码约定

## 设计说明

- **产物不入库**: `generated/` 与 `backups/` 是运行时产物，gitignored。
- **多目标**: 同一套模板可通过 `TINY_PROJECT_ROOT` 应用到不同仓库（主仓 core / gpui / flutter）。
- **历史遗留**: `registry.yaml` 的 `codebase.rust_path/core_flutter` 指向旧布局
  （`core_rust`/`core_flutter`），消费方以 `TINY_PROJECT_ROOT` 为准。
