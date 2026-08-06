# Flutter 模板使用指南

## 📋 概述

本目录包含 **Flutter 前端代码生成模板**，基于 **Clean Architecture + DDD** 架构模式，使用 Tera 模板引擎从 YAML 配置生成 Dart 代码。

## 🏗️ 架构层次

Flutter 模板遵循 Clean Architecture 的分层结构：

```
core_flutter/libs/features/{module}/{project}/
├── domain/                   # 领域层（纯业务逻辑）
│   ├── entities/            # 实体（core_flutter_domain_entity）
│   ├── repositories/        # 仓储接口（core_flutter_domain_repository）
│   └── value_objects/       # 值对象（core_flutter_domain_value_object）
├── application/             # 应用层（用例编排）
│   ├── use_cases/          # 用例（core_flutter_application_use_case）
│   ├── dto/                # 数据传输对象（core_flutter_application_dto）
│   ├── commands/           # 命令（core_flutter_application_command）
│   └── results/            # 结果（core_flutter_application_result）
├── infrastructure/          # 基础设施层（技术实现）
│   ├── repositories/       # 仓储实现（core_flutter_infrastructure_repository）
│   └── mappers/            # 映射器（core_flutter_infrastructure_mapper）
└── presentation/            # 表现层（UI）
    ├── providers/          # 状态管理（core_flutter_presentation_provider）
    └── screens/            # 界面（core_flutter_presentation_screen）
```

## 📦 模板列表

### Domain Layer（领域层）

#### 1. `core_flutter_domain_entity/v1`

**用途**: 生成领域实体（Domain Entity）

**模板文件**:
- `entity.tera` - 实体定义（8.5KB）
- `entity_mod.tera` - 模块导出

**特性**:
- ✅ 继承 `BaseEntity`（审计追踪、多租户）
- ✅ 丰富的业务方法（copyWith、toJson、fromJson）
- ✅ 值对象集成（Email、UserId、DisplayName 等）
- ✅ 不可变数据结构

**示例输出**:
```dart
class User extends BaseEntity {
  final UserId userId;
  final Email email;
  final DisplayName displayName;
  final bool isActive;
  
  // ... 构造函数、工厂方法、业务方法
}
```

#### 2. `core_flutter_domain_repository/v1`

**用途**: 生成领域层仓储接口（Repository Interface）

**模板文件**:
- `repository.tera` - 仓储接口（2.6KB）
- `repository_mod.tera` - 模块导出

**特性**:
- ✅ 抽象接口定义
- ✅ Result 类型（Either<Failure, T>）
- ✅ 标准 CRUD 方法（findAll、findById、create、update、delete）
- ✅ 多租户支持

**示例输出**:
```dart
abstract class UserRepository {
  Future<Result<List<User>, RepositoryFailure>> findAll({
    required TenantId tenantId,
    String? searchQuery,
    int? limit,
    int? offset,
  });
  
  Future<Result<User?, RepositoryFailure>> findById(UserId id, TenantId tenantId);
  Future<Result<User, RepositoryFailure>> create(User user);
  // ...
}
```

#### 3. `core_flutter_domain_value_object/v1`

**用途**: 生成值对象（Value Object）

**模板文件**:
- `value_object.tera` - 值对象定义（5.3KB）

**特性**:
- ✅ 不可变
- ✅ 内置验证逻辑
- ✅ 相等性比较（==、hashCode）
- ✅ toString() 实现

**示例输出**:
```dart
class DisplayName {
  final String value;
  
  DisplayName(this.value) {
    if (value.isEmpty) throw ValidationException('Display name cannot be empty');
  }
  
  @override
  bool operator ==(Object other) => other is DisplayName && value == other.value;
  
  @override
  int get hashCode => value.hashCode;
}
```

---

### Application Layer（应用层）

#### 4. `core_flutter_application_dto/v1`

**用途**: 生成数据传输对象（DTO）

**模板文件**:
- `dto.tera` - DTO 定义（5.0KB）

**特性**:
- ✅ 轻量级数据容器
- ✅ toJson/fromJson 序列化
- ✅ 分离输入输出 DTO（CreateUserDto、GetUserDto、UpdateUserDto 等）
- ✅ 验证方法

**示例输出**:
```dart
class CreateUserDto {
  final String tenantId;
  final String email;
  final String password;
  final String? displayName;
  
  CreateUserDto({
    required this.tenantId,
    required this.email,
    required this.password,
    this.displayName,
  });
  
  void validate() {
    if (email.isEmpty) throw ValidationException('Email is required');
    // ...
  }
}
```

#### 5. `core_flutter_application_use_case/v1`

**用途**: 生成用例（Use Case）

**模板文件**:
- `use_case.tera` - 标准用例（17.8KB）

**特性**:
- ✅ 单一职责原则
- ✅ Result 模式（Success/Failure）
- ✅ 异常处理
- ✅ 业务编排逻辑

**示例输出**:
```dart
class CreateUserUseCase extends CommandHandler<CreateUserCommand, CreateUserResult, Failure> {
  final UserRepository _repository;
  
  CreateUserUseCase(this._repository);
  
  @override
  Future<Result<CreateUserResult, Failure>> execute(CreateUserCommand command) async {
    try {
      command.validate();
      
      // 检查邮箱是否存在
      final emailExists = await _repository.existsByEmail(command.email.value, command.tenantId);
      if (emailExists.successOrNull ?? false) {
        return FailureResult(BusinessFailure(message: 'Email already exists'));
      }
      
      // 创建用户
      final user = User.create(/* ... */);
      final result = await _repository.create(user);
      
      return result.isSuccess 
        ? Success(CreateUserResult(result.successOrNull!))
        : FailureResult(InfrastructureFailure(/* ... */));
    } catch (e) {
      return FailureResult(/* ... */);
    }
  }
}
```

#### 6. `core_flutter_application_use_case_cqrs/v1`

**用途**: 生成 CQRS 风格用例（Command/Query 分离）

**模板文件**:
- `use_case.tera` - CQRS 用例（13.9KB）

**特性**:
- ✅ 命令查询分离（CQS）
- ✅ Command/Query 明确语义
- ✅ 事件发布机制
- ✅ 更细粒度的职责划分

#### 7. `core_flutter_application_command/v1`

**用途**: 生成命令对象（Command）

**模板文件**:
- `command.tera` - 命令定义（4.4KB）

**特性**:
- ✅ 命令模式
- ✅ 不可变
- ✅ 包含执行上下文（tenantId、userId 等）
- ✅ 验证逻辑

#### 8. `core_flutter_application_result/v1`

**用途**: 生成结果对象（Result）

**模板文件**:
- `result.tera` - 结果定义（1.3KB）

**特性**:
- ✅ 类型安全
- ✅ 成功/失败明确
- ✅ 携带业务数据或错误信息

---

### Infrastructure Layer（基础设施层）

#### 9. `core_flutter_infrastructure_repository/v1`

**用途**: 生成仓储实现（Repository Implementation）

**模板文件**:
- `repository_grpc.tera` - gRPC 仓储实现（11.8KB）
- `repository_mod.tera` - 模块导出

**特性**:
- ✅ gRPC 客户端集成（使用 contracts_dart）
- ✅ 异常转换（gRPC Exception → RepositoryFailure）
- ✅ 缓存支持（可选）
- ✅ 离线支持（可选）
- ✅ Proto 消息映射

**示例输出**:
```dart
class UserRepositoryImpl implements UserRepository {
  final UserServiceClient _client;
  
  UserRepositoryImpl(this._client);
  
  @override
  Future<Result<User?, RepositoryFailure>> findById(UserId id, TenantId tenantId) async {
    try {
      final request = GetUserRequest()
        ..id = id.value
        ..tenantId = tenantId.value;
        
      final response = await _client.getUser(request);
      final user = UserMapper.fromProto(response.user);
      
      return Success(user);
    } on GrpcError catch (e) {
      return FailureResult(RepositoryFailure(message: e.message));
    }
  }
}
```

#### 10. `core_flutter_infrastructure_mapper/v1`

**用途**: 生成数据映射器（Mapper）

**模板文件**:
- `mapper.tera` - 映射器（6.6KB）

**特性**:
- ✅ Proto ↔ Domain Entity 转换
- ✅ JSON ↔ Domain Entity 转换
- ✅ DTO ↔ Domain Entity 转换
- ✅ 空值处理

---

### Presentation Layer（表现层）

#### 11. `core_flutter_presentation_provider/v1`

**用途**: 生成状态管理（Provider/BLoC）

**模板文件**:
- `provider.tera` - Provider 定义（4.4KB）
- `provider_mod.tera` - 模块导出

**特性**:
- ✅ Riverpod 状态管理
- ✅ 加载状态管理（loading/error）
- ✅ 分页支持
- ✅ 错误处理

**示例输出**:
```dart
class UserState {
  final List<UserDto> users;
  final UserDto? selectedUser;
  final bool isLoading;
  final String? error;
  final int currentPage;
  final bool hasMore;
  
  UserState({/* ... */});
}

class UserNotifier extends StateNotifier<UserState> {
  final ListUsersUseCase _listUsersUseCase;
  final CreateUserUseCase _createUserUseCase;
  
  UserNotifier(this._listUsersUseCase, this._createUserUseCase) : super(UserState.initial());
  
  Future<void> loadUsers() async {
    state = state.copyWith(isLoading: true);
    
    final result = await _listUsersUseCase.execute(/* ... */);
    
    result.fold(
      (failure) => state = state.copyWith(error: failure.message, isLoading: false),
      (users) => state = state.copyWith(users: users, isLoading: false),
    );
  }
}
```

#### 12. `core_flutter_presentation_screen/v1`

**用途**: 生成界面组件（Screen）

**模板文件**:
- `screen_list.tera` - 列表界面（6.4KB）
- `screen_form.tera` - 表单界面（8.0KB）
- `screen_mod.tera` - 模块导出

**特性**:
- ✅ 响应式设计
- ✅ 下拉刷新
- ✅ 无限滚动
- ✅ 搜索栏
- ✅ 筛选选项
- ✅ Material Design 组件

**示例输出**:
```dart
class UserListScreen extends ConsumerWidget {
  const UserListScreen({Key? key}) : super(key: key);
  
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(userProvider);
    
    return Scaffold(
      appBar: AppBar(title: Text('用户管理')),
      body: state.isLoading
        ? CircularProgressIndicator()
        : ListView.builder(
            itemCount: state.users.length,
            itemBuilder: (context, index) {
              final user = state.users[index];
              return ListTile(
                title: Text(user.displayName),
                subtitle: Text(user.email),
                onTap: () => _navigateToDetail(context, user),
              );
            },
          ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _navigateToForm(context),
        child: Icon(Icons.add),
      ),
    );
  }
}
```

---

## 🚀 使用方法

### 1. 配置 YAML

在 `teras/modules/{module}/tasks/core_flutter.yaml` 中配置任务：

```yaml
tasks:
  - name: um_flutter_domain_entity
    description: "用户管理模块 Flutter Domain Entity 生成任务"
    template: "core_flutter_domain_entity/v1"
    template_file: "entity.tera"
    target: "lib/features/um/domain/entities/user.dart"
    path: "lib/features/um"
    
    features:
      freezed: true
      json_serializable: true
      equatable: false
```

### 2. 生成代码

```bash
# 生成单个任务
tiny tera gen --task um/flutter_domain_entity

# 生成所有 Flutter 任务
tiny tera gen --group um_flutter

# 查看差异
cat teras/generated/diff_report.json

# 应用到代码库
tiny tera apply --task um/flutter_domain_entity --interactive
```

### 3. 验证生成结果

```bash
# 检查 Flutter 代码
cd core_flutter
melos analyze

# 运行测试
melos test
```

---

## 🎯 命名约定

| 上下文 | 约定 | 示例 | Tera Filter |
|--------|------|------|-------------|
| Dart 类名 | `PascalCase` | `UserRepository` | `{{ entity.name \| pascal_case }}` |
| Dart 字段/方法 | `camelCase` | `displayName` | `{{ field.name \| camelCase }}` |
| Dart 文件名 | `snake_case` | `user_repository.dart` | `{{ entity.name \| snake_case }}` |
| Dart 常量 | `lowerCamelCase` | `kDefaultTimeout` | - |
| Dart 私有成员 | `_camelCase` | `_repository` | - |

---

## 📊 类型映射

元数据丰富器（`tiny_core::tera::enricher`）自动映射类型：

| YAML 类型 | Flutter 类型 | 示例 |
|-----------|--------------|------|
| `Uuid` | `String` | `'550e8400-e29b-41d4-a716-446655440000'` |
| `Email` | `Email` (VO) | `Email('user@example.com')` |
| `String` | `String` | `'Hello'` |
| `i32` | `int` | `42` |
| `i64` | `int` | `9223372036854775807` |
| `bool` | `bool` | `true` |
| `f32`/`f64` | `double` | `3.14` |
| `Timestamp` | `DateTime` | `DateTime.now().toUtc()` |
| `Vec<String>` | `List<String>` | `['admin', 'user']` |
| `Json` | `Map<String, dynamic>` | `{'key': 'value'}` |

---

## 🔍 手动代码保护

生成的代码支持**手动代码块保护**：

```dart
// 生成的文件
class User extends BaseEntity {
  // 自动生成的字段和方法
  
  // <<< block:custom_business_methods
  // 在这里添加自定义业务方法
  bool isEmailVerified() {
    return metadata['email_verified'] == true;
  }
  // >>> end:custom_business_methods
  
  // 自动生成的 copyWith、toString 等
}
```

重新生成时，`<<< block:xxx` 和 `>>> end:xxx` 之间的代码会被保留。

---

## 🛠️ 模板开发指南

### 创建新模板

1. **创建目录**: `teras/templates/core_flutter_{layer}_{type}/v1/`
2. **编写模板**: `your_template.tera`
3. **使用元数据变量**:
   ```jinja
   class {{ entity.name | pascal_case }} {
     {% for field in entity.fields %}
     final {{ field.flutter_type }} {{ field.name | camelCase }};
     {% endfor %}
   }
   ```
4. **配置任务**: 在 `tasks/core_flutter.yaml` 中添加任务定义
5. **测试**: `tiny tera gen --task your_task`

### 可用变量

所有模板都可访问 Rust 元数据丰富器注入的变量：

```jinja
{{ project }}                          # 项目名称（如 um）
{{ module }}                           # 模块名称（如 sys）
{{ entity.name }}                      # 实体名称
{{ entity.table_name }}                # 数据库表名
{{ entity.description }}               # 实体描述
{{ entity.fields }}                    # 字段列表
{{ field.name }}                       # 字段名
{{ field.flutter_type }}               # Flutter 类型（auto-enriched）
{{ field.nullable }}                   # 是否可空
{{ field.description }}                # 字段描述
{{ features.multi_tenant }}            # 是否多租户
{{ features.pagination }}              # 是否分页
```

---

## 📚 参考资料

- **Clean Architecture**: [core_flutter/libs/features/sys/um/](../../../../core_flutter/libs/features/sys/um/)
- **Rust 模板**: [core_rust_application_README.md](core_rust_application_README.md)
- **Metadata Enrichment**: [docs/rust_metadata.md](../../../../docs/rust_metadata.md)
- **命名规范**: [teras/templates/NAMING_CONVENTION.md](NAMING_CONVENTION.md)

---

## ✅ 最佳实践

1. **遵循 Clean Architecture** - 保持层次清晰，依赖方向正确（Domain ← Application ← Infrastructure ← Presentation）
2. **使用值对象** - 封装验证逻辑（Email、UserId 等）
3. **Result 模式** - 避免异常，使用 `Result<T, Failure>` 显式处理错误
4. **不可变数据** - 使用 `copyWith` 更新对象
5. **依赖注入** - 通过构造函数注入依赖
6. **测试优先** - 为生成的代码编写单元测试

---

## 🔄 更新流程

1. 修改 `teras/modules/um/um.yaml` 配置
2. 运行 `tiny tera gen --group um_flutter`
3. 检查 `teras/generated/diff_report.json`
4. 应用更改 `tiny tera apply --interactive`
5. 验证代码 `cd core_flutter && melos analyze`
6. 提交更改

---

**模板版本**: v1  
**最后更新**: 2025-01-02  
**维护者**: TinyDev Team
