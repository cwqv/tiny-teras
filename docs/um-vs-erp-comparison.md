# um vs erp 模板风格对比

> 日期：2026-08-12
> 状态：随 tera-erp-codegen-refactor G8 固化

## 为什么有两套风格

`um_domain`（默认）与 `erp_domain` 是两套**系统性冲突的代码约定**，源自消费侧不同模块的既有规范。模板按 profile 隔离，互不干扰。

## 差异对照

| 维度 | `um_domain`（默认） | `erp_domain` |
|---|---|---|
| 配置字段 | `profile` 缺省即 um | `profile: erp_domain` |
| 模板目录 | `templates/um/` | `templates/erp/` |
| 错误类型 | `shared_application::error` | `cqrs::error::{ApplicationError, ApplicationResult}` |
| ID 类型 | 裸 `Uuid` | `EntityId<T>` typed alias（`typed_id: true`） |
| 枚举 | 手写 `match`（from_str/as_str/display） | strum derives（Display/EnumString/Default）+ `from_db()` |
| Command/Handler | 分文件 | 合并单文件（product 风格） |
| DTO 三态 | 普通 `Option<T>` | `clearable: true` 字段用 `Option<Option<T>>` |
| Repository | 通用 CRUD | 乐观锁 `WHERE version = EXCLUDED.version` + `rows_affected()==0` |
| 同步声明 | — | `sync_strategy: NearTime/RealTime/LocalFirst` |

## 何时用哪个

- **um_domain**：`sys/*` 域（user/tenant/dict/config 等），既有代码风格稳定，向后兼容。
- **erp_domain**：`erp/*` 域（account/partner/product/sale 等），对齐 DDD + CQRS + typed-id 规范，是 tera 标准化的目标风格。

## 迁移注意

- um 模块迁移到 erp 模板**非强制**、按需。已有 um 模块保持旧模板向后兼容。
- 新增 ERP 实体优先用 erp 模板（`profile: erp_domain`）。
- 两套模板不可混用在一个模块内（profile 是模块级的）。
