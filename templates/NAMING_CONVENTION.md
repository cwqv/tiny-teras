# 模板命名规范

## 核心原则

1. **模板文件应该是通用的**：不绑定到特定实体，可被多个实体复用
2. **使用描述性的类型名称**：清晰表达模板的用途和层次
3. **保持简洁**：使用最短但清晰的名称

## 命名规范

### 模板文件命名格式

```
{type}.tera
```

其中 `{type}` 是模板类型，使用 snake_case，例如：
- `entity.tera` - 领域实体
- `repository.tera` - 仓储接口
- `service.tera` - 应用服务
- `dto.tera` - 数据传输对象
- `mod.tera` - 模块导出文件

### 目录结构

```
templates/
└── {language}_{layer}/
    └── v{version}/
        ├── {type}.tera
        └── {type}_mod.tera  # 模块文件使用 {type}_mod.tera
```

示例：
```
templates/
├── core_rust_domain_entity/
│   └── v1/
│       ├── entity.tera
│       └── entity_mod.tera
├── core_rust_domain_repository/
│   └── v1/
│       ├── repository.tera
│       └── repository_mod.tera
└── core_flutter_application_dto/
    └── v1/
        └── dto.tera
```

### 配置文件规范

在任务配置中：

```yaml
tasks:
  - name: um_domain_entity
    template: "core_rust_domain_entity/v1"
    template_file: "entity.tera"  # 使用通用模板名
    target: "crates/sys/um/src/domain/entities/user.rs"  # target 包含实体名
```

### 模板类型映射表

| 层次 | Rust 模板类型 | Flutter 模板类型 | 说明 |
|------|--------------|-----------------|------|
| Domain | `entity.tera` | `entity.tera` | 领域实体 |
| Domain | `repository.tera` | `repository.tera` | 仓储接口 |
| Infrastructure | `repository.tera` | `repository.tera` | 仓储实现 |
| Application | `service.tera` | `use_case.tera` | 应用服务/用例 |
| Application | `dto.tera` | `dto.tera` | 数据传输对象 |
| Presentation | - | `provider.tera` | 状态管理 |
| Presentation | - | `screen.tera` | UI 屏幕 |

### 特殊文件命名

- **模块文件**：使用 `{type}_mod.tera` 格式
  - `entity_mod.tera` - entities/mod.rs
  - `repository_mod.tera` - repositories/mod.rs
  - `service_mod.tera` - services/mod.rs

- **特定实现**：如果同一类型有多种实现，使用后缀区分
  - `repository_postgres.tera` - PostgreSQL 实现
  - `repository_grpc.tera` - gRPC 实现

### 目标文件命名

目标文件（target）应该包含实体名称，由模板变量生成：

```yaml
target: "crates/sys/um/src/domain/entities/{{ entity.name | snake_case }}.rs"
```

实际生成：
- `user.rs` - User 实体
- `tenant.rs` - Tenant 实体
- `user_repository.rs` - User 仓储

## 迁移指南

### 旧命名 → 新命名

| 旧名称 | 新名称 | 说明 |
|--------|--------|------|
| `domain_entity.tera` | `entity.tera` | 简化名称 |
| `user_entity.tera` | `entity.tera` | 移除实体特定名 |
| `repository_trait.tera` | `repository.tera` | 简化名称 |
| `application_service.tera` | `service.tera` | 简化名称 |
| `postgres_repository.tera` | `repository_postgres.tera` | 使用后缀区分实现 |

## 优势

1. **可复用性**：模板不绑定实体，可被多个实体使用
2. **清晰性**：名称简洁明了，易于理解
3. **一致性**：统一的命名规范，降低学习成本
4. **可扩展性**：易于添加新的模板类型

