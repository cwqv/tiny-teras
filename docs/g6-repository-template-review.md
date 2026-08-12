# G6 Review — repository_postgres.tera 模板评审

> 日期：2026-08-12
> 评审依据：account dry-run（66.2%）+ partner dry-run（62.4%）两次实测，两次暴露的 bug 均已修复。

## 一、模板能力评分

| 能力 | 状态 | 证据 |
|---|---|---|
| 乐观锁（WHERE version=EXCLUDED.version + version+1） | ✅ 可靠 | account/partner 生成一致 |
| 乐观锁冲突（rows_affected==0 → 错误） | ✅ 可靠 | 两实体生成一致 |
| EntityId 映射（PK/FK/枚举 from_db） | ✅ 可靠 | AccountId/TenantId/CurrencyId/UserId/枚举 |
| 关键字字段转义（r#type/r#ref + "type" 列引用） | ✅ 修复 | partner dry-run 暴露并修复 |
| 可选列（active/is_deleted/sync_version 分支） | ✅ 实测 | is_deleted 软删分支首次实测通过 |
| 嵌套宏设计（db_type/to_entity_expr/bind_expr） | ✅ 可维护 | 3 个宏职责清晰 |
| 保留块（<<< block:custom_methods） | ✅ 生效 | 生成文件含保留标记 |

## 二、覆盖率真实性评估

66.2%（account）/ 62.4%（partner）是**诚实指标**，因为：

- 覆盖的是**确定性骨架**（Row + to_entity + find_by_id + save + delete），这些是可安全生成的
- 未覆盖的是**动态逻辑**（list/count 动态 SQL、事务性 save、JOIN）——这些**不该**由模板生成（每种实体差异大，模板硬生成反而有害）
- 与 entity（70.2%）、application（110%+）互补，整体构成「骨架生成 + 业务手写」的合理分层

**结论**：覆盖率不是「生成度虚荣指标」，而是「可模板化的机械部分占比」。66.2%/62.4% 反映的是真实可自动化比例。

## 三、已知局限（未修复，接受为设计约束）

1. **事务性 save 不可模板化**：partner 的 save 用 `begin()` + sync_version MAX+1（D6 并发防撞），account 的 save 是单语句。模板只覆盖单语句 UPSERT——事务逻辑是实体内聚的，模板硬生成会产生误导性代码。
2. **动态过滤不可模板化**：list/count 的 filter 条件组合（active/type/search 等）每种实体不同，模板生成的通用版本反而难维护。
3. **嵌套宏的 entity 参数未用**：`db_type(entity, field)` 的 entity 参数实际未使用（字段级逻辑），但保留用于未来实体级类型推断，成本可忽略。

## 四、改进建议（未来）

| 建议 | 优先级 | 说明 |
|---|---|---|
| is_enum 精确匹配回归测试 | 高 | 本次修复（子串误匹配）应加单测防回归 |
| PartnerId/CountryId 等泛型 FK 通用化 | 中 | 当前按 rust_type contains 特判（CurrencyId/UserId），可改为「非标类型映射表」 |
| 事务性 save 的模板变体 | 低 | 若多实体需要事务，可做 `transactional_save: true` 配置分支 |

## 五、结论

**G6 模板可产线使用**。两次 dry-run（account 简单实体 + partner 复杂实体）覆盖了模板的核心路径，暴露的 2 个真实 bug（is_enum 污染、关键字转义）已修复且回归验证（account 66.2% 不变、um 零错误）。模板价值与边界都清晰：确定性骨架生成 + 动态逻辑手写。

**产出**：`repository_postgres.tera`（245→280 行，含修复）+ `tera-compare-account-repository.md`（66.2%）+ `tera-compare-partner-repository.md`（62.4%）。
