# Tera 代码生成对比报告 — account Repository

**模块**: account (account.accounts)
**模板**: templates/erp/core_rust_infrastructure_repository/v1/repository_postgres.tera
**生成时间**: 2026-08-12

## 总体统计

- **总体覆盖率**: 66.2%
- **生成总行数**: 202
- **手工总行数**: 305
- **目标覆盖率**: 35%（G6，40/120 行）→ **实际 66.2%，超出目标 +31.2pt**

## 文件明细

### postgres_account_repository.rs

- 生成: 202 行
- 手工: 305 行
- 覆盖率: 66.2%

## 生成覆盖的方法

| 方法 | 覆盖 |
|---|---|
| `AccountRow` struct（FromRow，19 字段） | ✅ 完整生成 |
| `AccountRow::to_entity()`（EntityId 映射） | ✅ 完整生成（AccountId/TenantId/CurrencyId/UserId/枚举 from_db） |
| `PostgresAccountRepository` struct + `new()` | ✅ 完整生成 |
| `find_by_id()` | ✅ 完整生成（typed id bind） |
| `save()`（UPSERT + 乐观锁） | ✅ 完整生成（WHERE version = EXCLUDED.version + rows_affected==0 冲突检查） |
| `delete()`（归档 active=false） | ✅ 完整生成（可选列：有 active 无 is_deleted → 归档语义） |

## 手工残留（不在模板范围）

- `find_by_code()`：按 code_store 精确查询（业务方法）
- `list()`：动态过滤 SQL（active/account_type/internal_group/search 条件 + 分页）
- `count()`：动态计数 SQL

## 模板关键能力验证（G6.1-6.6）

- **乐观锁**：`WHERE account.accounts.version = EXCLUDED.version` + `version = account.accounts.version + 1` ✅
- **冲突检查**：`rows_affected() == 0 → 乐观锁冲突错误` ✅
- **EntityId 映射**：Row→Entity `AccountId::from_uuid().expect("non-nil DB PK")`；Entity→bind `account.id.as_uuid()`；FK `CurrencyId::from_uuid().expect("non-nil DB FK")` ✅
- **可选列**：无 sync_version → 不生成 find_since/get_max_sync_version；有 active 无 is_deleted → `active = false` 归档 ✅
- **枚举处理**：`account_type: Option<String>`（DB）↔ `AccountType::from_db`（Entity），经 enricher mark_enum_fields 修复（rust_type 泛型包裹匹配 + 显式 rust_enum_name） ✅
- **repo_crate 前缀**：`erp_account::domain::entities::account::{...}`（server crate 引用 core 类型） ✅

## 实施中发现并修复的引擎问题

1. **enricher mark_enum_fields 枚举匹配缺陷**（`core/tools/dev_core/src/tera/enricher.rs`）：
   - `enum_names` 只收集 enum `name`（如 `account_type`），`rust_type`（`Option<AccountType>`）精确匹配失败 → `is_enum=false`
   - `enum_rust_name` 用 `to_pascal_case(field_type)` 推导，忽略 yaml 显式 `rust_enum_name`（enricher v2 保留的 `enum_name`）
   - 修复：收集 `enum_rust_name` 集合匹配 `rust_type.contains()`；优先用 `enum_name` 字段
2. **TaskConfig 缺 repo 相关字段**：新增 `repo_crate`（server 侧 crate 前缀）+ `repo_entity_module`（模块名覆盖）
3. **Tera 循环内 `{% set %}` 不逃逸**：has_active/has_sync_version 等标志位改用 `{% set_global %}`

## 结论

G6 目标达成且超预期：repository 模板覆盖率 66.2%（目标 35%）。模板已具备产线可用性，可推广到 partner/product 的扁平实体 repo。
