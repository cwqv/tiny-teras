# ERP Module YAML Schema 定义说明
#
# 每个 ERP 模块在 dev_infra/teras/modules/erp/ 下有独立的 YAML 文件。
# 文件结构:
#
# module:
#   name: erp_sale              # crate 名称
#   schema: sale                # PostgreSQL schema
#   display_name: "Sales"       # 显示名称
#   description: "..."          # 模块描述
#
# entities:
#   - name: SaleOrder
#     table: sale.orders
#     aggregate_root: true       # 是否为聚合根 (影响事件生成)
#     parent: null               # 父实体 (用于 header-line 关系)
#     parent_fk: null
#     features:
#       soft_delete: true
#       optimistic_locking: true
#       tenant_aware: true
#       sync_support: true
#     fields:
#       - name: id
#         type: Uuid
#         rust_type: "uuid::Uuid"
#         primary_key: true
#       - name: state
#         type: Enum
#         rust_type: "String"
#         enum: SaleOrderState
#         default: Draft
#       - name: amount_total
#         type: Money
#         rust_type: "rust_decimal::Decimal"
#       - name: partner_id
#         type: Uuid
#         rust_type: "Option<uuid::Uuid>"
#         relation: base.partners
#       - name: product_uom_qty
#         type: f64
#         rust_type: "f64"
#     enums:
#       SaleOrderState: [Draft, Confirmed, Locked, Done, Cancel]
#     computed_fields:           # Money 类型字段触发 computed 模块生成
#       - amount_untaxed
#       - amount_tax
#       - amount_total
#
# task_groups:
#   backend:
#     - domain_entity
#     - domain_repository
#     - infrastructure_repository
#     - application_command
#     - application_query
#     - application_dto
#     - application_service
#   proto:
#     - proto_service
#   frontend:
#     - flutter_list_page
#     - flutter_form_page
#     - flutter_detail_page
