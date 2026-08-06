# Proto 模板使用指南

## 📋 概述

本目录包含 **Protocol Buffers (Proto) 代码生成模板**，用于生成 gRPC 服务定义、消息类型和请求/响应结构。使用 Tera 模板引擎从 YAML 配置自动生成 `.proto` 文件。

## 🏗️ 模板结构

```
core_proto/v1/
├── proto_complete.tera         # 完整 Proto 文件（推荐）
├── proto_service.tera           # 服务定义（遗留）
├── proto_message.tera           # 消息定义
└── proto_request_response.tera  # 请求/响应定义
```

## 📦 模板列表

### 1. `proto_complete.tera` ✅ **推荐使用**

**用途**: 生成完整的 `.proto` 文件，包含服务、消息、请求和响应

**特性**:
- ✅ 完整的 gRPC 服务定义
- ✅ 实体消息（Entity Message）
- ✅ 标准 CRUD 方法（Get, List, GetPaged, Create, Update, Delete, BatchDelete）
- ✅ 所有请求/响应消息
- ✅ 自动 proto 字段编号
- ✅ 多租户支持
- ✅ 分页和搜索支持

**生成示例**:
```protobuf
syntax = "proto3";

package tinydev.v1.um;

import "tinydev/v1/common.proto";

// ============ UserService ============

service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc GetUsersPaged(GetUsersPagedRequest) returns (GetUsersPagedResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (Empty);
  rpc BatchDeleteUsers(BatchDeleteUsersRequest) returns (BatchDeleteUsersResponse);
}

// ============ User Entity ============

message User {
  string id = 1;  // 用户 ID
  string email = 2;  // 邮箱
  string display_name = 3;  // 显示名称
  bool is_active = 4;  // 是否激活
  // ... 其他字段
}

// ============ GetUserRequest ============

message GetUserRequest {
  string id = 1;
  string tenant_id = 2;
}

// ... 其他请求/响应消息
```

**YAML 配置**:
```yaml
- name: um_proto_complete
  description: "UM Proto 完整服务生成"
  template: "core_proto/v1"
  template_file: "proto_complete.tera"
  target: "protos/tinydev/v1/sys/um.proto"
  
  proto_package: "tinydev.v1.um"
  
  features:
    multi_tenant: true
    pagination: true
    search: true
  
  searchable_fields:
    - email
    - display_name
```

---

### 2. `proto_message.tera`

**用途**: 单独生成 Proto 消息定义

**特性**:
- ✅ 实体消息生成
- ✅ 自动过滤隐藏字段（如 `encrypted_password`）
- ✅ 自动 proto 字段编号
- ✅ 字段注释

**生成示例**:
```protobuf
// ============ User Message ============

message User {
  string id = 1;  // 用户 ID
  string email = 2;  // 邮箱
  string display_name = 3;  // 显示名称
  repeated string roles = 4;  // 角色列表
  bool is_active = 5;  // 是否激活
}
```

**用途场景**:
- 生成嵌套消息
- 生成 DTO 消息
- 单独维护消息定义

---

### 3. `proto_request_response.tera`

**用途**: 生成请求和响应消息

**特性**:
- ✅ 标准 CRUD 请求/响应
- ✅ 分页请求/响应
- ✅ 批量操作请求/响应
- ✅ 多租户支持
- ✅ 搜索过滤支持

**生成示例**:
```protobuf
// ============ CreateUserRequest ============

message CreateUserRequest {
  string tenant_id = 1;
  string email = 2;
  string password = 3;
  string display_name = 4;
  repeated string roles = 5;
}

// ============ CreateUserResponse ============

message CreateUserResponse {
  User user = 1;
}

// ============ ListUsersRequest ============

message ListUsersRequest {
  string tenant_id = 1;
  string search_query = 2;  // Search across: email, display_name
  bool is_active = 3;
}

// ... 其他消息
```

---

### 4. `proto_service.tera`（遗留）

**用途**: 生成服务定义（保留向后兼容）

**推荐**: 使用 `proto_complete.tera` 替代

---

## 🚀 使用方法

### 1. 配置 YAML

在 `teras/modules/{module}/tasks/protos.yaml` 中配置任务：

```yaml
tasks:
  - name: um_proto_complete
    description: "UM Proto 完整服务生成"
    template: "core_proto/v1"
    template_file: "proto_complete.tera"
    target: "protos/tinydev/v1/sys/um.proto"
    path: "protos/tinydev/v1/sys"
    
    proto_package: "tinydev.v1.um"
    
    features:
      multi_tenant: true
      pagination: true
      search: true
    
    searchable_fields:
      - email
      - display_name
      - roles
```

### 2. 生成 Proto 文件

```bash
# 生成单个任务
tiny tera gen --task um/proto_complete

# 生成所有 Proto 任务
tiny tera gen --group um_proto

# 查看差异
cat teras/generated/diff_report.json

# 应用到代码库
tiny tera apply --task um/proto_complete --interactive
```

### 3. 编译 Proto 文件

```bash
# 使用 Buf 编译
cd protos
buf generate

# 生成 Rust gRPC 代码到 contracts/
# 生成 Dart gRPC 代码到 contracts_dart/
```

---

## 🎯 隐藏字段机制

某些字段不应暴露在 gRPC API 中（如密码、内部元数据），可通过 `hidden_fields` 配置隐藏：

**YAML 配置**:
```yaml
# teras/modules/um/um.yaml
entities:
  - name: User
    hidden_fields:
      - encrypted_password  # 不在 Proto 中生成
      - raw_user_meta_data  # 不在 Proto 中生成
```

**生成的 Proto**:
```protobuf
message User {
  string id = 1;
  string email = 2;
  // encrypted_password 被隐藏
  string display_name = 3;
  // raw_user_meta_data 被隐藏
}
```

---

## 📊 类型映射

元数据丰富器（`tiny_core::tera::enricher`）自动映射类型：

| YAML 类型 | Proto 类型 | 示例 |
|-----------|-----------|------|
| `Uuid` | `string` | `'550e8400-e29b-41d4-a716-446655440000'` |
| `String` | `string` | `'Hello'` |
| `Email` | `string` | `'user@example.com'` |
| `i32` | `int32` | `42` |
| `i64` | `int64` | `9223372036854775807` |
| `bool` | `bool` | `true` |
| `f32` | `float` | `3.14` |
| `f64` | `double` | `3.14159265359` |
| `Timestamp` | `string` | `'2025-01-01T12:00:00Z'`（ISO 8601） |
| `Vec<String>` | `repeated string` | `['admin', 'user']` |
| `Json` | `string` | `'{"key":"value"}'`（JSON 字符串） |

---

## 🔍 Service Methods 配置

可自定义生成的 gRPC 方法：

**YAML 配置**:
```yaml
# um.yaml
entities:
  - name: User
    service_methods:
      - Get        # 生成 GetUser
      - List       # 生成 ListUsers
      - Create     # 生成 CreateUser
      - Update     # 生成 UpdateUser
      - Delete     # 生成 DeleteUser
      # 不生成 GetPaged 和 BatchDelete
```

**可用方法**:
- `Get` - 获取单个实体
- `List` - 获取列表（无分页）
- `GetPaged` - 获取分页列表
- `Create` - 创建实体
- `Update` - 更新实体
- `Delete` - 删除实体
- `BatchDelete` - 批量删除

---

## 🛠️ 高级配置

### 自定义 Proto 导入

```yaml
entities:
  - name: User
    proto_imports:
      - "google/protobuf/timestamp.proto"
      - "tinydev/v1/common.proto"
```

### 自定义 Proto Package

```yaml
tasks:
  - name: um_proto_complete
    proto_package: "com.mycompany.v1.um"
```

### 多租户支持

```yaml
features:
  multi_tenant: true  # 自动添加 tenant_id 到所有请求
```

### 搜索字段配置

```yaml
searchable_fields:
  - email
  - display_name
  - roles  # 生成 ListUsersRequest 时包含这些字段作为过滤器
```

---

## 📚 最佳实践

1. **使用 proto_complete.tera**
   - 一个文件包含完整服务定义
   - 便于维护和理解

2. **隐藏敏感字段**
   - 使用 `hidden_fields` 保护密码、令牌等
   - 避免暴露内部实现细节

3. **版本化 Proto Package**
   - 使用 `v1`, `v2` 等版本号
   - 便于 API 演进和向后兼容

4. **遵循 Proto 命名规范**
   - 服务名：`PascalCase` + `Service`（`UserService`）
   - 消息名：`PascalCase`（`CreateUserRequest`）
   - 字段名：`snake_case`（`display_name`）

5. **使用 Buf**
   - 使用 `buf lint` 检查代码风格
   - 使用 `buf format` 格式化 proto 文件
   - 使用 `buf generate` 生成代码

---

## 🔄 工作流示例

```bash
# 1. 修改实体配置
vim teras/modules/um/um.yaml

# 2. 生成 Proto 文件
tiny tera gen --task um/proto_complete

# 3. 查看差异
cat teras/generated/diff_report.json

# 4. 应用更改
tiny tera apply --task um/proto_complete

# 5. 编译 Proto
cd protos
buf lint
buf generate

# 6. 验证生成的 Rust 代码
cd ../core_rust
cargo build

# 7. 验证生成的 Dart 代码
cd ../core_flutter
melos bootstrap
melos analyze

# 8. 提交更改
git add .
git commit -m "feat(proto): regenerate um service"
```

---

## ✅ 验证清单

- [ ] Proto 文件语法正确（`buf lint`）
- [ ] 所有消息有正确的字段编号
- [ ] 隐藏字段未出现在 Proto 中
- [ ] Service 包含必要的 RPC 方法
- [ ] 多租户请求包含 `tenant_id`
- [ ] 分页请求包含 `page`, `page_size`, `sort_by`
- [ ] `buf generate` 成功生成 Rust 和 Dart 代码
- [ ] 生成的代码可编译

---

## 📖 参考资料

- **Proto 规范**: [Protocol Buffers Language Guide](https://protobuf.dev/programming-guides/proto3/)
- **Buf 文档**: [Buf Documentation](https://buf.build/docs/)
- **gRPC 最佳实践**: [gRPC Best Practices](https://grpc.io/docs/guides/best-practices/)
- **Rust gRPC**: [Tonic Documentation](https://github.com/hyperium/tonic)
- **Dart gRPC**: [grpc-dart](https://github.com/grpc/grpc-dart)

---

**模板版本**: v1  
**最后更新**: 2025-01-02  
**维护者**: TinyDev Team
