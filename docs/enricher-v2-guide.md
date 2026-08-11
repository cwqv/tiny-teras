# MetadataEnricher V2 — 自定义字段使用指南

## 概述

MetadataEnricher V2 引入了 `extra_config` 机制，允许模块配置文件（如 `account.yaml`）中定义自定义字段，并在 Tera 模板中访问这些字段。这使得 ERP 域模板可以访问 `typed_id`、`sync_strategy`、`clearable` 等 ERP 特有的配置，而不破坏 um 域的向后兼容性。

## 背景

**问题**：旧版 MetadataEnricher 只保留预定义字段（`name`、`table_name`、`fields` 等），自定义字段在 enricher 处理时被丢弃。这导致 ERP 模板无法访问 `typed_id`、`sync_strategy` 等配置。

**解决方案**：使用 serde 的 `#[serde(flatten)]` 特性，将所有非预定义字段保存到 `extra_config: HashMap<String, serde_yaml::Value>` 中，并传递到模板上下文。

## 核心变化

### EntityConfig 和 FieldConfig 结构

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EntityConfig {
    // 预定义字段
    pub name: String,
    pub table_name: String,
    pub fields: Vec<FieldConfig>,
    // ...其他预定义字段...
    
    /// 额外配置字段（捕获所有自定义字段）
    #[serde(flatten)]
    pub extra_config: HashMap<String, serde_yaml::Value>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FieldConfig {
    // 预定义字段
    pub name: String,
    pub r#type: String,
    // ...其他预定义字段...
    
    /// 额外配置字段（捕获字段级自定义字段）
    #[serde(flatten)]
    pub extra_config: HashMap<String, serde_yaml::Value>,
}
```

## 在 YAML 配置中定义自定义字段

### Entity 级自定义字段

```yaml
# account.yaml
entities:
  - name: Account
    table_name: accounts
    schema: account
    
    # ERP 自定义字段
    typed_id: true                    # 是否生成 type AccountId = EntityId<Account>
    sync_strategy: RealTime           # 同步策略：RealTime | NearTime | LocalFirst
    timestamp_style: separated        # 时间戳风格：separated | unified
    
    fields:
      - name: id
        type: uuid
        rust_type: Uuid
```

### Field 级自定义字段

```yaml
fields:
  - name: description
    type: string
    rust_type: String
    nullable: true
    
    # 字段级自定义字段
    clearable: true                   # 是否可清空（三态 Option）
    timestamp_style: unified          # 字段级时间戳风格
```

## 在 Tera 模板中访问自定义字段

### 访问 Entity 自定义字段

```tera
{# templates/erp/entity.tera #}

{% if entity.extra_config.typed_id %}
// 生成类型别名
pub type {{ entity.name }}Id = EntityId<{{ entity.name }}>;
{% endif %}

{% if entity.extra_config.sync_strategy %}
// 生成同步策略
impl Command for {{ entity.name }}Command {
    fn sync_strategy() -> SyncStrategy {
        SyncStrategy::{{ entity.extra_config.sync_strategy }}
    }
}
{% endif %}
```

### 访问 Field 自定义字段

```tera
{# templates/erp/dto.tera #}

{% for field in entity.fields %}
    {% if field.extra_config.clearable %}
    // 三态 Option：None=不变更、Some(None)=清空、Some(Some(v))=赋新值
    pub {{ field.name }}: Option<Option<{{ field.rust_type }}>>,
    {% else %}
    // 普通 Option：None=不变更、Some(v)=赋新值
    pub {{ field.name }}: Option<{{ field.rust_type }}>,
    {% endif %}
{% endfor %}
```

### 类型转换

`extra_config` 中的值是 `serde_yaml::Value`，在 Tera 模板中可以直接使用：

- **布尔值**：`{% if entity.extra_config.typed_id %}` 直接判断真假
- **字符串**：`{{ entity.extra_config.sync_strategy }}` 直接输出
- **数字**：`{{ field.extra_config.min_length }}` 直接输出

## 已知自定义字段

以下是 ERP 域常用的自定义字段（仅供参考，模板作者可自由扩展）：

### Entity 级

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `typed_id` | bool | 是否生成 `type XxxId = EntityId<Xxx>` | `true` |
| `sync_strategy` | string | CQRS 同步策略 | `RealTime` / `NearTime` / `LocalFirst` |
| `timestamp_style` | string | 时间戳风格 | `separated` / `unified` |

### Field 级

| 字段 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `clearable` | bool | 是否可清空（生成三态 Option） | `true` |
| `timestamp_style` | string | 字段级时间戳风格覆盖 | `unified` |

## 未知自定义字段处理

**关键特性**：未知自定义字段会原样传递到模板，**不验证、不报错**。

这意味着：
- 模板作者可以自由定义新的自定义字段，无需修改 MetadataEnricher 代码
- 如果模板中访问不存在的字段（如 `entity.extra_config.nonexistent`），Tera 会返回空值（falsy）

### 示例：自定义验证规则

```yaml
# account.yaml
fields:
  - name: code
    type: string
    rust_type: String
    
    # 自定义验证规则（模板自定义）
    custom_validation: "alphanumeric"
    min_length: 3
    max_length: 20
```

在模板中：

```tera
{% if field.extra_config.custom_validation %}
// 自定义验证
match {{ field.extra_config.custom_validation }} {
    "alphanumeric" => validate_alphanumeric(&self.{{ field.name }}),
    _ => Ok(()),
}
{% endif %}
```

## 向后兼容性

**关键保证**：um 域模块不使用自定义字段，因此不受影响。

- `modules/um/um.yaml` 没有定义任何自定义字段 → `extra_config` 为空 HashMap → 模板中访问 `entity.extra_config.xxx` 返回空值 → um 模板正常工作
- 回归测试：`tinyd tera gen --task um --dry-run` 生成输出与之前一致

## 迁移指南

### 从旧模板迁移到 V2

如果你的旧模板需要访问自定义字段：

1. **修改 YAML 配置**：在 `entity` 或 `field` 级添加自定义字段
2. **修改 Tera 模板**：使用 `entity.extra_config.xxx` 或 `field.extra_config.xxx` 访问
3. **测试**：使用 `--dry-run` 生成并对比输出

### 示例：添加 typed_id 支持

**修改前**（旧模板）：

```tera
{# entity.tera #}
pub struct {{ entity.name }} {
    pub id: Uuid,
    // ...
}
```

**修改后**（V2 模板）：

```tera
{# templates/erp/entity.tera #}
{% if entity.extra_config.typed_id %}
pub type {{ entity.name }}Id = EntityId<{{ entity.name }}>;
{% endif %}

pub struct {{ entity.name }} {
    {% if entity.extra_config.typed_id %}
    pub id: {{ entity.name }}Id,
    {% else %}
    pub id: Uuid,
    {% endif %}
    // ...
}
```

**YAML 配置**：

```yaml
entities:
  - name: Account
    typed_id: true  # 启用类型化 ID
    fields: [...]
```

## 最佳实践

1. **文档化自定义字段**：在模板目录的 README 中列出支持的自定义字段及其含义
2. **提供默认值**：使用 `{% if entity.extra_config.xxx %}...{% else %}...{% endif %}` 处理字段不存在的情况
3. **保持简单**：不要在 YAML 中嵌套复杂对象，优先使用 bool/string/number
4. **测试覆盖**：为每个自定义字段编写单元测试（见 `core/tools/dev_core/src/tera/config.rs` 的测试）

## 示例：完整的 ERP Entity 模板

```tera
{# templates/erp/entity.tera #}
use serde::{Deserialize, Serialize};
{% if entity.extra_config.typed_id %}
use kernel::types::entity_id::EntityId;
{% else %}
use uuid::Uuid;
{% endif %}
use strum_macros::{Display, EnumString};
use cqrs::error::{ApplicationError, ApplicationResult};

{% if entity.extra_config.typed_id %}
pub type {{ entity.name }}Id = EntityId<{{ entity.name }}>;
{% endif %}

/// {{ entity.description | default(value=entity.name + " 实体") }}
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct {{ entity.name }} {
    {% for field in entity.fields %}
    {% if field.name == "id" and entity.extra_config.typed_id %}
    pub id: {{ entity.name }}Id,
    {% else %}
    pub {{ field.name }}: {{ field.rust_type }},
    {% endif %}
    {% endfor %}
}

impl {{ entity.name }} {
    // 业务方法留在 custom_methods 块手工编写
}

#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_new_defaults() {
        // 生成基础测试骨架
    }
}
```

## 故障排查

### 问题：模板中访问 extra_config.xxx 返回空

**原因**：YAML 配置中未定义该字段。

**解决**：在 `account.yaml` 的 `entity` 或 `field` 级添加该字段。

### 问题：生成后发现 typed_id 未生效

**原因**：旧版 MetadataEnricher 丢弃了自定义字段。

**解决**：确保使用 V2 版本（包含 `extra_config` 字段），运行单元测试验证：

```bash
cargo test --manifest-path core/Cargo.toml -p dev_core --lib tera::config::tests::test_entity_extra_config_typed_id
```

### 问题：um 模块生成失败

**原因**：V2 破坏了向后兼容性。

**解决**：运行回归测试：

```bash
cargo run --manifest-path dev_infra/tools/dev_cli/Cargo.toml -p dev_cli --bin tinyd -- tera gen --task um --dry-run
```

如果失败，检查是否在 um 模板中误用了 `extra_config`。

## 参考

- **单元测试**：`core/tools/dev_core/src/tera/config.rs` 的 `tests` 模块
- **ERP 模板示例**：`tiny-teras/templates/erp/entity.tera`（待创建）
- **um 模板参考**：`tiny-teras/templates/um/entity.tera`（向后兼容）
- **设计文档**：`openspec/changes/tera-erp-codegen-refactor/design.md`

## 变更历史

- **2026-08-11**：初始版本，支持 `typed_id`、`sync_strategy`、`clearable`、`timestamp_style`
- 未来可扩展：更多 ERP 特有字段，无需修改 enricher 代码
