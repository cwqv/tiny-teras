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

## Profile 机制

模板按 profile 隔离为两套风格（`templates/um/` 与 `templates/erp/`），模块配置通过 `profile:` 字段选择：

```yaml
# modules/<module>/<module>.yaml
project: account
profile: erp_domain   # erp_domain → templates/erp/，um_domain(默认) → templates/um/
```

模板查找优先级：**模块本地**（`modules/<m>/templates/`）→ **自定义覆盖**（模块级 `templates: {task: "custom/..."}`，G7.5）→ **profile 默认** → **全局通用**。

参考：
- [Profile 系统使用指南](docs/profile-system-guide.md)
- [um vs erp 模板风格对比](docs/um-vs-erp-comparison.md)

## ERP 模板（erp_domain）

面向 DDD + CQRS 规范（typed-id / strum 枚举 / 乐观锁 / Command+Handler 合一），当前家族：

| 家族 | 产出 |
|---|---|
| `core_rust_domain_entity/v1` | 实体 + typed-id + 枚举 + new/validate + 测试骨架 |
| `core_rust_application_dto/v1` | Dto / Create / Update（三态 clearable）/ Query / List |
| `core_rust_application_command/v1` | Command + Handler 合一（cqrs::ApplicationError + SyncStrategy） |
| `core_rust_application_query/v1` | Query + Handler 合一 |
| `core_rust_infrastructure_repository/v1` | PG repo 骨架（乐观锁 / EntityId 映射 / 可选列） |

参考：[ERP 模板使用指南](docs/erp-template-guide.md)

## 覆盖率量化（tera compare）

`tinyd tera compare --module <m> --file <f>` 量化生成覆盖率，支持 `--json` 与 `--min-coverage`（CI 回归检测）。

参考：[compare 工具用户手册](docs/compare-tool-guide.md)

## 修改模板

1. 编辑 `templates/**/*.tera` 或 `modules/**/*.yaml`
2. 提交 PR 到本仓，各消费方拉取后生效
3. 模板与目标代码结构强相关——修改模板需同步修改消费方的代码约定

## 设计说明

- **产物不入库**: `generated/` 与 `backups/` 是运行时产物，gitignored。
- **多目标**: 同一套模板可通过 `TINY_PROJECT_ROOT` 应用到不同仓库（主仓 core / gpui / flutter）。
- **历史遗留**: `registry.yaml` 的 `codebase.rust_path/core_flutter` 指向旧布局
  （`core_rust`/`core_flutter`），消费方以 `TINY_PROJECT_ROOT` 为准。
- **覆盖率实测**（2026-08-12）：account 模块 entity 70.2%、application 75-80%、repository 66.2%
  （目标 entity 55% / application 75% / repository 35%，见 `tera-compare-*.md`）。
