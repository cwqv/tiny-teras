# Profile 系统使用指南

## 概述

Profile 系统允许不同的模块使用不同风格的 Tera 模板集。目前支持两个 profile：

- **`um_domain`**：用户管理域风格（默认），使用 `shared_application::error`、裸 Uuid、手写枚举
- **`erp_domain`**：ERP 域风格，使用 `cqrs::ApplicationError`、`EntityId<T>`、strum 枚举

## 背景

**问题**：um 域与 ERP 域的代码风格系统性冲突：
- 错误类型：`shared_application::error` vs `cqrs::error`
- ID 类型：裸 `Uuid` vs `EntityId<T>`
- 枚举处理：手写 `match` vs `strum` derive
- 文件组织：分立 Command/Handler vs 合一文件

**解决方案**：Profile 系统根据模块配置自动选择对应的模板集，实现模板隔离。

## 如何使用 Profile

### 在模块配置中指定 Profile

在模块的 YAML 配置文件（如 `modules/account/account.yaml`）中添加 `profile` 字段：

```yaml
# account.yaml
project: account
version: v1
profile: erp_domain  # 使用 ERP 域模板

entities:
  - name: Account
    typed_id: true
    sync_strategy: NearTime
    # ...
```

**可选值**：
- `erp_domain` — 使用 `templates/erp/` 下的模板
- `um_domain` — 使用 `templates/um/` 下的模板（默认）

如果省略 `profile` 字段，默认使用 `um_domain` 保持向后兼容。

### 运行生成时的输出

当运行 `tinyd tera gen --task account` 时，会显示活跃的 profile：

```
🚀 开始生成任务: account
📋 项目: account
📋 版本: v1
📋 描述: 会计科目（account.accounts）模块
🎨 Profile: erp_domain (使用模板: templates/erp/*)
```

## 模板查找顺序

Profile 系统实现了三层模板查找机制（按优先级从高到低）：

1. **模块本地模板**：`modules/{module}/templates/{task_template}/`
   - 用于模块特定的定制模板
   - 最高优先级，覆盖全局模板

2. **Profile 模板**：`templates/{profile_subdir}/{task_template}/`
   - 根据模块 `profile` 字段选择
   - `erp_domain` → `templates/erp/`
   - `um_domain` → `templates/um/`

3. **通用模板（向后兼容）**：`templates/{task_template}/`
   - 用于没有 profile 分类的旧模板
   - 最低优先级

### 示例

假设模块配置 `profile: erp_domain`，任务模板为 `core_rust_application_command/v1/command.tera`：

查找顺序：
1. `modules/account/templates/core_rust_application_command/v1/command.tera`（模块本地）
2. `templates/erp/core_rust_application_command/v1/command.tera`（profile 模板）
3. `templates/core_rust_application_command/v1/command.tera`（通用模板）

找到第一个存在的模板后停止查找。

## ERP Domain Profile 特性

使用 `profile: erp_domain` 时，模板支持以下特性：

### 1. typed_id 支持

```yaml
entities:
  - name: Account
    typed_id: true  # 生成 type AccountId = EntityId<Account>
```

生成的代码：
```rust
use kernel::types::entity_id::EntityId;

pub type AccountId = EntityId<Account>;

pub struct Account {
    pub id: AccountId,  // 类型化 ID，非裸 Uuid
    // ...
}
```

### 2. CQRS 错误类型

```rust
use cqrs::error::{ApplicationError, ApplicationResult};

// 验证错误
return Err(ApplicationError::validation(vec!["字段不能为空".to_string()]));

// 未找到
return Err(ApplicationError::not_found("实体不存在"));

// 冲突
return Err(ApplicationError::conflict("版本冲突"));
```

### 3. SyncStrategy 声明

```yaml
entities:
  - name: Account
    sync_strategy: NearTime  # RealTime / NearTime / LocalFirst
```

生成的代码：
```rust
impl Command for CreateAccountCommand {
    fn sync_strategy(&self) -> SyncStrategy {
        SyncStrategy::NearTime
    }
}
```

### 4. 三态 Option（clearable 字段）

```yaml
fields:
  - name: description
    type: String
    nullable: true
    clearable: true  # 可清空字段
```

生成的 UpdateDto：
```rust
pub struct UpdateAccountDto {
    /// 三态 Option: None=不变更, Some(None)=清空, Some(Some(v))=赋新值
    pub description: Option<Option<String>>,
}
```

### 5. 乐观锁支持

如果 entity 有 `version` 字段，Update handler 自动包含：

```rust
entity.version += 1;  // 更新版本号
self.repository.update(&entity).await?;  // repository 层检查乐观锁
```

### 6. Command + Handler 合一

ERP 模板将 Command 和 Handler 生成在同一个文件中，符合 ERP 实际代码风格。

## UM Domain Profile 特性

使用 `profile: um_domain`（或默认）时：

- 使用 `shared_application::error`
- 使用裸 `Uuid`
- Command 和 Handler 分立两个文件
- 枚举使用手写 `match` 块
- 向后兼容旧代码风格

## 如何添加新 Profile

如果未来需要添加新的 profile（如 `crm_domain`、`hr_domain`），按以下步骤操作：

### 1. 创建模板目录

```bash
mkdir -p templates/crm
```

### 2. 在模板目录中创建模板

将模板文件放在 `templates/crm/` 下，保持与现有结构一致：

```
templates/crm/
├── command.tera
├── query.tera
├── dto.tera
└── entity.tera
```

### 3. 更新 profile 解析函数

编辑 `dev_infra/tools/dev_cli/src/tera_commands.rs`，在 `resolve_profile_template_dir` 函数中添加新 profile：

```rust
fn resolve_profile_template_dir(profile: &str) -> Result<&str> {
    match profile {
        "erp_domain" => Ok("erp"),
        "um_domain" => Ok("um"),
        "crm_domain" => Ok("crm"),  // 新增
        _ => Err(anyhow::anyhow!(
            "未知的 profile: {}。仅支持 erp_domain、um_domain、crm_domain",
            profile
        )),
    }
}
```

### 4. 更新配置文件

在模块配置中使用新 profile：

```yaml
project: customer
version: v1
profile: crm_domain  # 使用 CRM 域模板
```

### 5. 测试

```bash
cargo run --manifest-path dev_infra/tools/dev_cli/Cargo.toml -p dev_cli --bin tinyd -- tera gen --task customer --dry-run
```

确认输出显示正确的 profile：
```
🎨 Profile: crm_domain (使用模板: templates/crm/*)
```

## 最佳实践

1. **明确指定 profile**：即使使用默认 `um_domain`，也建议在配置中显式声明 `profile: um_domain`，提高可读性。

2. **保持模板集完整性**：每个 profile 下的模板应该是自洽的，避免依赖其他 profile 的模板。

3. **文档化 profile 差异**：在模板目录中创建 README，说明该 profile 的特性和适用场景。

4. **避免过度定制**：优先使用 profile 模板，只在必要时使用模块本地模板覆盖。

5. **测试向后兼容**：添加新 profile 后，运行回归测试确保旧模块不受影响：
   ```bash
   cargo run --manifest-path dev_infra/tools/dev_cli/Cargo.toml -p dev_cli --bin tinyd -- tera gen --task um --dry-run
   ```

## 故障排查

### 问题：模板未找到

**错误信息**：`模板不存在: core_rust_application_command/v1/command.tera`

**原因**：指定的 profile 下没有对应的模板。

**解决**：
1. 检查模板文件是否存在于 `templates/{profile_subdir}/`
2. 检查模块配置中的 `profile` 字段是否正确
3. 检查任务配置中的 `template` 和 `template_file` 是否匹配

### 问题：Profile 验证失败

**错误信息**：`未知的 profile: xxx。仅支持 erp_domain 和 um_domain`

**原因**：配置文件中指定了不支持的 profile。

**解决**：修改 `profile` 字段为 `erp_domain` 或 `um_domain`，或按照"如何添加新 Profile"步骤添加新 profile。

### 问题：um 模块生成失败

**原因**：旧模板没有移动到 `templates/um/` 目录。

**解决**：
```bash
cd tiny-teras/templates
mkdir -p um
mv core_rust_* um/
mv core_flutter_* um/
mv core_proto um/
```

## 迁移指南

### 从旧版本迁移到 Profile 系统

如果你的模块配置文件没有 `profile` 字段：

1. **无需修改**：默认使用 `um_domain`，向后兼容。
2. **可选优化**：在配置中添加 `profile: um_domain`，提高明确性。
3. **ERP 模块迁移**：如果是 ERP 域模块，添加 `profile: erp_domain` 并添加 `typed_id`、`sync_strategy` 等字段。

### 示例：um 模块（无需修改）

```yaml
# modules/um/um.yaml
project: um
version: v1
# profile 字段省略，默认 um_domain
entities:
  - name: User
    # ...
```

### 示例：account 模块（迁移到 ERP profile）

```yaml
# modules/account/account.yaml
project: account
version: v1
profile: erp_domain  # 新增

entities:
  - name: Account
    typed_id: true           # 新增
    sync_strategy: NearTime  # 新增
    fields:
      - name: description
        clearable: true      # 新增（可选字段）
        # ...
```

## 参考

- **ERP 模板示例**：`tiny-teras/templates/erp/`
- **UM 模板参考**：`tiny-teras/templates/um/`
- **实现代码**：`dev_infra/tools/dev_cli/src/tera_commands.rs` 中的 `resolve_profile_template_dir` 函数
- **配置结构**：`core/tools/dev_core/src/tera/config.rs` 中的 `TeraConfig`
- **MetadataEnricher V2 指南**：`tiny-teras/docs/enricher-v2-guide.md`

## 变更历史

- **2026-08-11**：初始版本，支持 `erp_domain` 和 `um_domain` 两个 profile
- 未来可扩展：更多领域 profile（crm_domain、hr_domain 等）
