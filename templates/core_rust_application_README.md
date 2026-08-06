# Rust Application Layer 模板说明

本目录包含 Rust 应用层的 Tera 模板，用于生成 CQRS 风格的应用层代码。

## 📁 模板结构

```
core_rust_application_*/
├── dto/                  # 数据传输对象
│   ├── dto_complete.tera    # 完整的 DTO 定义（Create/Update/Delete/Get/List）
│   └── mapper.tera          # DTO ↔ Command/Query 映射器
├── command/              # 命令
│   ├── create_command.tera  # Create 命令
│   ├── update_command.tera  # Update 命令
│   └── delete_command.tera  # Delete 命令
├── query/                # 查询
│   ├── get_query.tera       # Get 查询
│   └── list_query.tera      # List 查询
├── command_handler/      # 命令处理器
│   └── create_handler.tera  # Create 命令处理器
├── query_handler/        # 查询处理器
│   ├── get_handler.tera     # Get 查询处理器
│   └── list_handler.tera    # List 查询处理器
└── service/              # 应用服务
    └── service_complete.tera # 完整的应用服务
```

## 🎯 使用方式

### 1. 生成 DTO

**模板**: `core_rust_application_dto/v1/dto_complete.tera`

**输出**: `crates/sys/{module}/src/application/dto/{entity}_dto.rs`

**包含**:
- `Create{Entity}Dto` - 创建实体的 DTO
- `Created{Entity}Dto` - 创建成功响应 DTO
- `Get{Entity}Dto` - 查询单个实体的 DTO
- `List{Entity}sDto` - 查询列表的 DTO
- `Update{Entity}Dto` - 更新实体的 DTO
- `Delete{Entity}Dto` - 删除实体的 DTO

### 2. 生成 DTO Mapper

**模板**: `core_rust_application_dto/v1/mapper.tera`

**输出**: `crates/sys/{module}/src/application/mappers/{entity}_dto_mapper.rs`

**功能**:
- DTO → Command 转换
- DTO → Query 转换
- 验证 Email 等值对象

### 3. 生成 Command

**模板**:
- `core_rust_application_command/v1/create_command.tera`
- `core_rust_application_command/v1/update_command.tera`
- `core_rust_application_command/v1/delete_command.tera`

**输出**: `crates/sys/{module}/src/application/commands/{action}_{entity}_command.rs`

### 4. 生成 Query

**模板**:
- `core_rust_application_query/v1/get_query.tera`
- `core_rust_application_query/v1/list_query.tera`

**输出**: `crates/sys/{module}/src/application/queries/{action}_{entity}_query.rs`

### 5. 生成 Command Handler

**模板**: `core_rust_application_command_handler/v1/create_handler.tera`

**输出**: `crates/sys/{module}/src/application/handlers/create_{entity}_command_handler.rs`

**功能**:
- 业务规则验证
- 密码哈希（如果是 User/Credential 实体）
- 实体创建
- 事件发布

### 6. 生成 Query Handler

**模板**:
- `core_rust_application_query_handler/v1/get_handler.tera`
- `core_rust_application_query_handler/v1/list_handler.tera`

**输出**: `crates/sys/{module}/src/application/handlers/{action}_{entity}_query_handler.rs`

### 7. 生成 Application Service

**模板**: `core_rust_application_service/v1/service_complete.tera`

**输出**: `crates/sys/{module}/src/application/services/{entity}_service.rs`

**功能**:
- 聚合所有 handlers
- 提供统一的服务接口
- 简化调用者的使用

## 🔧 元数据字段说明

### 必需字段

- `entity.name` - 实体名称（如 "User"）
- `entity.fields` - 字段列表
- `entity.schema` - 数据库 schema（如 "sys"）
- `entity.table_name` - 表名

### 字段属性

每个 field 需要：
- `name` - 字段名
- `rust_type` - Rust 类型（如 `uuid::Uuid`, `String`, `shared_domain::value_objects::Email`）
- `description` - 字段描述
- `primary_key` - 是否为主键（boolean）
- `nullable` - 是否可空（boolean）
- `searchable` - 是否可搜索（boolean，可选）

### 可选字段

- `entity.hidden_fields` - 隐藏字段列表（如 `["password_hash"]`）
- `entity.description` - 实体描述

## 📝 示例配置

```yaml
# teras/modules/um/um.yaml
entities:
  - name: User
    table_name: users
    schema: sys
    description: "用户实体"
    
    hidden_fields:
      - encrypted_password
    
    fields:
      - name: id
        type: Uuid
        rust_type: "uuid::Uuid"
        nullable: false
        primary_key: true
        description: "用户 ID"
      
      - name: email
        type: Email
        rust_type: "shared_domain::value_objects::Email"
        nullable: false
        unique: true
        description: "邮箱"
        searchable: true
      
      - name: encrypted_password
        type: String
        rust_type: "String"
        nullable: false
        description: "加密密码"
      
      - name: tenant_id
        type: String
        rust_type: "String"
        nullable: false
        description: "租户 ID"
```

## 🎨 模板变量

### 全局变量

- `{{ entity.name }}` - 实体名（原始）
- `{{ entity.name | snake_case }}` - 蛇形命名（如 `user_credential`）
- `{{ entity.name | pascal_case }}` - 帕斯卡命名（如 `UserCredential`）
- `{{ entity.description }}` - 实体描述

### 字段变量

- `{{ field.name }}` - 字段名
- `{{ field.name | snake_case }}` - 蛇形命名
- `{{ field.rust_type }}` - Rust 类型
- `{{ field.description }}` - 字段描述
- `{{ field.primary_key }}` - 是否主键
- `{{ field.nullable }}` - 是否可空

### 条件判断

```jinja
{% if entity.schema == "sys" or entity.table_name contains "tenant" %}
    // 多租户相关代码
{% endif %}

{% if entity.name == "User" or entity.name contains "Credential" %}
    // 用户认证相关代码
{% endif %}

{% for field in entity.fields %}
{% if field.rust_type contains "Email" %}
    // Email 特殊处理
{% endif %}
{% endfor %}
```

## 🔐 手写代码保护

所有模板都包含 **block 标记**，用于保护手写代码：

```rust
// <<< block:custom_service_methods
// 在此添加自定义方法，重新生成时会保留
pub async fn activate_user(&self, id: Uuid) -> ApplicationResult<()> {
    // 自定义业务逻辑
}
// >>> end:custom_service_methods
```

**重要**：只在 block 内编写自定义代码，block 外的代码会在重新生成时覆盖。

## ✅ 最佳实践

1. **DTO vs Command/Query**
   - DTO 用于 API 层（gRPC/REST）
   - Command/Query 用于应用层
   - Mapper 负责转换和验证

2. **Handler 职责**
   - 业务规则验证
   - 调用 Repository
   - 发布领域事件

3. **Service 职责**
   - 聚合 Handlers
   - 提供简洁的 API
   - 不包含业务逻辑

4. **事件发布**
   - 使用 `UnifiedEventService`
   - 异步发布，不阻塞主流程
   - 失败只记录日志，不影响业务

## 🚀 下一步

- [ ] 添加 Update/Delete Handler 模板
- [ ] 添加分页查询模板
- [ ] 添加批量操作模板
- [ ] 添加事务处理模板

---

**维护**: 请在修改模板后更新此文档
