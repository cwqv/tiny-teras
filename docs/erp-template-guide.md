# ERP 模板使用指南

> 日期：2026-08-12
> 状态：随 tera-erp-codegen-refactor G8 固化

本指南说明如何用 ERP 域模板（`templates/erp/`）生成标准 DDD 模块。以 `account.accounts` 为示例。

## 1. 模块配置（account.yaml）

```yaml
project: account
version: v1
description: "会计科目（account.accounts）模块"
profile: erp_domain  # 使用 ERP 域模板（templates/erp/*）

entities:
  - name: Account
    table_name: accounts
    schema: account
    typed_id: true              # 触发 `type AccountId = EntityId<Account>`
    sync_strategy: NearTime     # 声明 CQRS 同步策略

    service_methods: [Get, List, Create, Update, Delete]

    fields:
      - { name: id, type: Uuid, rust_type: "AccountId", primary_key: true }
      - { name: tenant_id, type: String, rust_type: "kernel::multi_tenant::TenantId" }
      - { name: account_type, type: String, rust_type: "Option<AccountType>", is_enum: true, enum_name: AccountType }
      ...
```

## 2. 任务配置（tasks/core_rust.yaml）

每个生成文件对应一个 task，`template` 指向模板家族，`template_file` 指向具体模板：

```yaml
tasks:
  - name: account_domain_entity
    template: "core_rust_domain_entity/v1"
    template_file: "entity.tera"
    target: "core/domains/erp/account/src/domain/entities/account_account.rs"

  # PG Repository 骨架（server 侧实现）
  - name: account_repository_postgres
    template: "core_rust_infrastructure_repository/v1"
    template_file: "repository_postgres.tera"
    target: "server/modules/erp/erp_account_server/src/infrastructure/repositories/postgres_account_repository.rs"
    repo_crate: "erp_account"   # 生成文件内 core 类型的 crate 引用前缀
```

## 3. 生成与应用

```bash
# 生成到 generated/（dry-run，不碰源码）
tinyd tera gen --group account

# 量化覆盖率（CI 可用 --min-coverage 回归检测）
tinyd tera compare --module account --file repository
tinyd tera compare --module account --json --min-coverage 35

# 应用到源码
tinyd tera apply --group account
```

## 4. 生成覆盖 vs 手工残留

| 层 | 模板生成 | 手工残留 |
|---|---|---|
| entity | struct + typed-id + strum 枚举 + new() + validate 骨架 + 测试骨架 | 业务方法（`archive`/`internal_group` 映射等）、复杂校验 |
| dto | Dto / Create / Update（三态 clearable）/ Query / List | — |
| command/query | Command + Handler 合一文件 | 复杂业务编排（如 post/cancel 序列） |
| repository | Row + to_entity + find_by_id + save(乐观锁) + delete + 可选 sync | find_by_code / list / count（JOIN、聚合） |
| 手写（不走模板） | — | Port trait、use_cases facade、gRPC handler、mod.rs 注册 |

## 5. 保留块（block preservation）

模板输出含 `<<< block:custom_methods` / `>>> end:custom_methods` 标记。重新生成时，标记之间的手工代码会被保留。不要在标记外手写需要保留的逻辑。

## 6. 模板覆盖（G7.5）

模块可覆盖 profile 内特定模板：

```yaml
profile: erp_domain
templates:
  "core_rust_domain_entity/v1": "custom/entity_v2"
```

解析优先级：模块本地 → 自定义覆盖（`templates/{profile}/{path}` 或 `templates/{path}`）→ profile 默认 → 全局通用。
