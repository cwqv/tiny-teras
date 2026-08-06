# Tiny Tera 模板规划文档

## 已完成模板

### 1. core_rust_dto/v1
- **用途**: 生成 Rust DTO (Data Transfer Objects)
- **文件**: 
  - `dto_file_header.tera` - 文件头部导入
  - `dto_create.tera` - Create DTO
  - `dto_created.tera` - Created Response DTO
  - `dto_get.tera` - Get DTO
  - `dto_list.tera` - List DTO
  - `dto_update.tera` - Update DTO
  - `dto_delete.tera` - Delete DTO

### 2. core_sql_migration/v1
- **用途**: 生成 SQL Migration 文件
- **文件**:
  - `migration_create_table.tera` - 创建表的 Migration

### 3. core_proto/v1
- **用途**: 生成 Protobuf 定义文件
- **文件**:
  - `proto_service.tera` - gRPC Service 和 Message 定义

## Core Rust UM Crate 需要建立的模板

### 1. core_rust_domain_entity/v1
- **用途**: 生成 Domain Entity (领域实体)
- **文件**:
  - `domain_entity.tera` - 领域实体定义
  - `domain_entity_mod.tera` - domain/entities/mod.rs

### 2. core_rust_domain_repository/v1
- **用途**: 生成 Domain Repository Trait (仓储接口)
- **文件**:
  - `repository_trait.tera` - Repository Trait 定义
  - `repository_mod.tera` - domain/repositories/mod.rs

### 3. core_rust_infrastructure_repository/v1
- **用途**: 生成 Infrastructure Repository Implementation (仓储实现)
- **文件**:
  - `postgres_repository.tera` - PostgreSQL 仓储实现
  - `repository_mod.tera` - infrastructure/repositories/mod.rs

### 4. core_rust_application_service/v1
- **用途**: 生成 Application Service (应用服务)
- **文件**:
  - `application_service.tera` - 应用服务实现
  - `service_mod.tera` - application/services/mod.rs

## Core Flutter 需要建立的模板和 YAML

### 架构说明

Core Flutter 当前没有基于 DDD Clean Architecture 实现，需要建立以下结构：

```
libs/features/sys/um/
├── domain/
│   ├── entities/
│   │   └── user_credential.dart
│   ├── repositories/
│   │   └── user_credential_repository.dart
│   └── value_objects/
│       └── email.dart
├── application/
│   ├── use_cases/
│   │   ├── create_user_use_case.dart
│   │   ├── get_user_use_case.dart
│   │   ├── list_users_use_case.dart
│   │   ├── update_user_use_case.dart
│   │   └── delete_user_use_case.dart
│   └── dto/
│       └── user_dto.dart
├── infrastructure/
│   ├── repositories/
│   │   └── grpc_user_credential_repository.dart
│   └── data_sources/
│       └── grpc_user_data_source.dart
└── presentation/
    ├── providers/
    │   └── user_providers.dart
    ├── screens/
    │   ├── user_list_screen.dart
    │   └── user_form_screen.dart
    └── widgets/
        └── user_card.dart
```

### 模板集规划

#### 1. core_flutter_domain_entity/v1
- **用途**: 生成 Flutter Domain Entity
- **文件**:
  - `domain_entity.tera` - 领域实体类
  - `domain_entity_mod.tera` - domain/entities/mod.dart

#### 2. core_flutter_domain_repository/v1
- **用途**: 生成 Flutter Domain Repository Interface
- **文件**:
  - `repository_interface.tera` - Repository 接口
  - `repository_mod.tera` - domain/repositories/mod.dart

#### 3. core_flutter_application_use_case/v1
- **用途**: 生成 Flutter Use Cases
- **文件**:
  - `use_case_create.tera` - Create Use Case
  - `use_case_get.tera` - Get Use Case
  - `use_case_list.tera` - List Use Case
  - `use_case_update.tera` - Update Use Case
  - `use_case_delete.tera` - Delete Use Case
  - `use_case_mod.tera` - application/use_cases/mod.dart

#### 4. core_flutter_application_dto/v1
- **用途**: 生成 Flutter DTO
- **文件**:
  - `dto_file_header.tera` - 文件头部导入
  - `dto_create.tera` - Create DTO
  - `dto_get.tera` - Get DTO
  - `dto_list.tera` - List DTO
  - `dto_update.tera` - Update DTO
  - `dto_delete.tera` - Delete DTO

#### 5. core_flutter_infrastructure_repository/v1
- **用途**: 生成 Flutter Infrastructure Repository (gRPC)
- **文件**:
  - `grpc_repository.tera` - gRPC Repository 实现
  - `grpc_data_source.tera` - gRPC Data Source
  - `repository_mod.tera` - infrastructure/repositories/mod.dart

#### 6. core_flutter_presentation_provider/v1
- **用途**: 生成 Flutter Presentation Providers (Riverpod)
- **文件**:
  - `providers.tera` - Riverpod Providers
  - `providers_mod.tera` - presentation/providers/mod.dart

#### 7. core_flutter_presentation_screen/v1
- **用途**: 生成 Flutter Presentation Screens
- **文件**:
  - `list_screen.tera` - List Screen
  - `form_screen.tera` - Form Screen (Create/Update)
  - `screens_mod.tera` - presentation/screens/mod.dart

### YAML 配置规划

#### um_flutter.yaml
- **位置**: `teras/projects/um/um_flutter.yaml`
- **结构**: 类似 `um.yaml`，但针对 Flutter 生成
- **任务**:
  - `um_flutter_domain_entity` - 生成 Domain Entity
  - `um_flutter_domain_repository` - 生成 Repository Interface
  - `um_flutter_application_use_cases` - 生成 Use Cases
  - `um_flutter_application_dto` - 生成 DTOs
  - `um_flutter_infrastructure_repository` - 生成 gRPC Repository
  - `um_flutter_presentation_providers` - 生成 Providers
  - `um_flutter_presentation_screens` - 生成 Screens

## 实施优先级

### Phase 1: Core Rust 模板（高优先级）
1. ✅ core_rust_dto/v1 (已完成)
2. ⏳ core_rust_domain_entity/v1
3. ⏳ core_rust_domain_repository/v1
4. ⏳ core_rust_infrastructure_repository/v1
5. ⏳ core_rust_application_service/v1

### Phase 2: Core Flutter 模板（中优先级）
1. ⏳ core_flutter_domain_entity/v1
2. ⏳ core_flutter_domain_repository/v1
3. ⏳ core_flutter_application_use_case/v1
4. ⏳ core_flutter_application_dto/v1
5. ⏳ core_flutter_infrastructure_repository/v1
6. ⏳ core_flutter_presentation_provider/v1
7. ⏳ core_flutter_presentation_screen/v1

## 注意事项

1. **类型映射**: 需要建立 Rust 类型到 Flutter 类型的映射表
2. **依赖管理**: Flutter 模板需要考虑 shared 库的依赖
3. **代码风格**: Flutter 代码需要遵循 Dart 代码规范
4. **测试**: 每个模板生成后需要验证生成的代码质量

## Workspace 模块约束

- `workspace/task` 和后续 `workspace/memo` 使用嵌套模块路径：`modules/workspace/<module>/<module>.yaml`
- `teras` 只应生成可复制的模块骨架，不应覆盖 sync、conflict resolution、复杂产品交互等手工逻辑
- `workspace/task` 作为当前标准参考模块，优先验证 module path、task pack、runtime registration 的闭环
- `core_workspace_module/v1/` 目前提供最小模板：`module_descriptor.tera`、`module_registration.tera`、`flutter_panel.tera`
