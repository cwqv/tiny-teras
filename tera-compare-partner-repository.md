# Tera 代码生成对比报告 — partner Repository

**模块**: partner (base.partners)
**模板**: templates/erp/core_rust_infrastructure_repository/v1/repository_postgres.tera
**生成时间**: 2026-08-12

## 总体统计

- **总体覆盖率**: 62.4%
- **生成总行数**: 338
- **手工总行数**: 542
- **对照**: account 66.2%（partner 是更复杂实体，62.4% 达标）

## 文件明细

### postgres_partner_repository.rs

- 生成: 338 行
- 手工: 542 行
- 覆盖率: 62.4%

## dry-run 验证结论（任务 2）

partner 作为 **G6 模板的复杂实体压力测试**，验证了模板在以下场景的表现：

| 场景 | 结果 |
|---|---|
| 42 字段大实体（vs account 19） | ✅ 模板正常处理 |
| self-FK `PartnerId`（parent_id / commercial_partner_id） | ✅ 作为 Uuid 存储，Entity 侧保持 Option<PartnerId> |
| `PartnerType` 枚举（from_db） | ✅ 正确 |
| **关键字字段 `type` / `ref`** | ✅ 修复后生成 `r#type` / `r#ref`（struct + 访问 + bind） |
| `f64`（经纬度）/ `i32`（color）/ `bool` 混合 | ✅ 正常 |
| **is_deleted 软删分支**（模板 has_is_deleted 首次实测） | ✅ `SET is_deleted = true, updated_at = NOW()` |
| `has_sync_version` 分支 | ⚠️ 未触发（partner 实体无 sync_version 字段，sync_version 是 repo 内部字段） |
| 事务性 save（手写版 D6 并发防撞 MAX+1） | ❌ 模板覆盖不了（预期内，事务逻辑手写） |

## 手工残留（不在模板范围）

- **事务性 save**：手写版用 `begin()` + sync_version `MAX+1` 计算（D6 并发防撞），模板无法生成
- `find_by_code`（按 ref 查询）
- `list` / `search`（JOIN base.countries 取 country_name + 动态过滤）
- `count`（动态计数）
- `apply_partner_operation`（NearTime LWW 合并，sync_arm 层）

## 模板改进（dry-run 暴露并已修复）

1. **is_enum 子串误匹配污染**（enricher，严重）：枚举名 `"type"` 通过 `rust_type.contains()` 误匹配 `kernel::types::user_id::UserId` 中的 `"types"`，导致 user_id/created_by/updated_by 全被标记为枚举并生成 `PartnerType::from_db`。
   - 修复：改为提取 `rust_type` 顶层类型（去 Option/Vec 包裹）后**精确匹配**枚举名集合 + 显式 `enum_name` 字段判定。
2. **关键字字段缺失转义**：`type`/`ref` 生成裸标识符（`pub type` 非法）。
   - 修复：enricher 新增 `rust_field_name`（关键字 → `r#type`）+ `sql_column_name`（关键字 → `"type"` 双引号 PG 引用）；模板 Rust 标识符用 rust_field_name、SQL 列用 sql_column_name。
3. **回归验证**：account 覆盖率 66.2% 不变、um 零错误（模板改动向后兼容）。

## 结论

partner dry-run 达成目标：**实测 62.4% 覆盖率（目标 35% 的 1.8 倍）**，暴露并修复了 2 个模板引擎真实 bug（is_enum 污染 + 关键字转义）。模板在复杂实体（42 字段/self-FK/软删/枚举/关键字字段）上验证可产线使用。事务性 save 与 JOIN 查询为固有手工残留，符合设计预期。
