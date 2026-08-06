# SQL Migration 模板使用指南

## 📋 概述

本目录包含 **SQL 数据库迁移模板**，用于生成 PostgreSQL DDL 语句。使用 Tera 模板引擎从 YAML 配置自动生成安全、可回滚的迁移脚本。

## 🏗️ 模板结构

```
core_sql_migration/v1/
├── migration_create_table.tera   # CREATE TABLE 迁移
├── migration_alter_table.tera     # ALTER TABLE 迁移
├── migration_drop_table.tera      # DROP TABLE 迁移
└── migration_constraints.tera     # 约束迁移（FK/UK/CK/Index）
```

## 📦 模板列表

### 1. `migration_create_table.tera` ✅ **最常用**

**用途**: 生成 CREATE TABLE 迁移脚本

**特性**:
- ✅ 完整的表定义（列、类型、约束）
- ✅ 主键、外键、唯一约束
- ✅ 检查约束（CHECK）
- ✅ 索引（B-Tree、GIN、GiST）
- ✅ 注释（COMMENT ON TABLE/COLUMN）
- ✅ 多租户支持
- ✅ 审计字段（created_at、updated_at、version）
- ✅ 软删除支持
- ✅ 回滚脚本注释

**生成示例**:
```sql
-- ============================================================
-- Create sys.users
-- File: 20251217000112_create_sys_users.sql
-- Purpose:
--   - Create User entity table under schema `sys`
-- Dependencies:
--   - 20251217000101_create_sys_schema.sql
--   - 20251001000001_create_extensions.sql
-- ============================================================

BEGIN;

CREATE SCHEMA IF NOT EXISTS sys;

SET LOCAL search_path = sys, public, extensions;

-- Table: sys.users
CREATE TABLE IF NOT EXISTS sys.users (
  id                  UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  email               CITEXT      NOT NULL,
  encrypted_password  TEXT        NOT NULL,
  display_name        TEXT        NULL,
  is_active           BOOLEAN     NOT NULL DEFAULT TRUE,
  is_locked           BOOLEAN     NOT NULL DEFAULT FALSE,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  version             INTEGER     NOT NULL DEFAULT 1,
  tenant_id           TEXT        NOT NULL,
  
  -- Constraints
  CONSTRAINT users_email_format_chk CHECK (length(email) >= 3 AND length(email) <= 320),
  CONSTRAINT users_password_nonempty_chk CHECK (length(encrypted_password) >= 20)
);

-- Unique indexes
CREATE UNIQUE INDEX IF NOT EXISTS ux_sys_users_email ON sys.users (email);

-- Common query indexes
CREATE INDEX IF NOT EXISTS ix_sys_users_is_active ON sys.users (is_active);
CREATE INDEX IF NOT EXISTS ix_sys_users_tenant_id ON sys.users (tenant_id);

-- Comments
COMMENT ON TABLE sys.users IS '用户实体';
COMMENT ON COLUMN sys.users.email IS '邮箱';

COMMIT;

-- ============================================================
-- Rollback (manual)
-- ============================================================
-- BEGIN;
-- DROP TABLE IF EXISTS sys.users;
-- COMMIT;
```

**YAML 配置**:
```yaml
- name: um_migration_create_table
  description: "UM CREATE TABLE Migration"
  template: "core_sql_migration/v1"
  template_file: "migration_create_table.tera"
  target: "migrations/{{ migration_timestamp }}_create_sys_users.sql"
  
  features:
    multi_tenant: true
```

---

### 2. `migration_alter_table.tera`

**用途**: 生成 ALTER TABLE 迁移脚本（修改表结构）

**特性**:
- ✅ 添加列（ADD COLUMN）
- ✅ 删除列（DROP COLUMN）
- ✅ 重命名列（RENAME COLUMN）
- ✅ 修改列类型（ALTER COLUMN TYPE）
- ✅ 添加/删除约束（ADD/DROP CONSTRAINT）
- ✅ 添加/删除索引（CREATE/DROP INDEX）
- ✅ 手动代码块保护（`<<< block:custom_pre_migration >>>`）
- ✅ 回滚脚本生成

**生成示例**:
```sql
-- ============================================================
-- Alter sys.users
-- File: 20251218000001_alter_sys_users.sql
-- Purpose:
--   - Modify User entity table structure
--   - Add avatar_url column
--   - Rename raw_user_meta_data to metadata
-- ============================================================

BEGIN;

SET LOCAL search_path = sys, public, extensions;

-- <<< block:custom_pre_migration
-- Add custom pre-migration SQL here
-- >>> end:custom_pre_migration

-- ------------------------------------------------------------
-- Add Columns
-- ------------------------------------------------------------
ALTER TABLE sys.users
  ADD COLUMN avatar_url TEXT NULL;
COMMENT ON COLUMN sys.users.avatar_url IS '头像URL';

ALTER TABLE sys.users
  ADD COLUMN phone TEXT NULL;

-- ------------------------------------------------------------
-- Rename Columns
-- ------------------------------------------------------------
ALTER TABLE sys.users
  RENAME COLUMN raw_user_meta_data TO metadata;

-- ------------------------------------------------------------
-- Modify Column Types
-- ------------------------------------------------------------
ALTER TABLE sys.users
  ALTER COLUMN display_name TYPE VARCHAR(255);

-- ------------------------------------------------------------
-- Add Constraints
-- ------------------------------------------------------------
ALTER TABLE sys.users
  ADD CONSTRAINT users_phone_format_chk CHECK (phone IS NULL OR length(phone) >= 7);

-- ------------------------------------------------------------
-- Add Indexes
-- ------------------------------------------------------------
CREATE INDEX IF NOT EXISTS idx_users_phone ON sys.users (phone);

-- <<< block:custom_post_migration
-- Add custom post-migration SQL here
-- >>> end:custom_post_migration

COMMIT;

-- ============================================================
-- Rollback (manual)
-- ============================================================
-- BEGIN;
-- ALTER TABLE sys.users DROP COLUMN IF EXISTS avatar_url;
-- ALTER TABLE sys.users DROP COLUMN IF EXISTS phone;
-- ALTER TABLE sys.users RENAME COLUMN metadata TO raw_user_meta_data;
-- COMMIT;
```

**YAML 配置**:
```yaml
- name: um_migration_alter_table
  description: "UM ALTER TABLE Migration"
  template: "core_sql_migration/v1"
  template_file: "migration_alter_table.tera"
  target: "migrations/{{ migration_timestamp }}_alter_sys_users.sql"
  
  migration_description: "Add avatar_url and phone columns"
  
  alterations:
    add_columns:
      - name: avatar_url
        type: String
        sql_type: TEXT
        nullable: true
        description: "头像URL"
      
      - name: phone
        type: String
        sql_type: TEXT
        nullable: true
    
    rename_columns:
      - old_name: raw_user_meta_data
        new_name: metadata
    
    modify_columns:
      - name: display_name
        new_sql_type: VARCHAR(255)
    
    add_constraints:
      - name: users_phone_format_chk
        expression: "phone IS NULL OR length(phone) >= 7"
    
    add_indexes:
      - name: idx_users_phone
        columns:
          - phone
```

---

### 3. `migration_drop_table.tera` ⚠️ **谨慎使用**

**用途**: 生成 DROP TABLE 迁移脚本（删除表）

**特性**:
- ⚠️ 破坏性操作警告
- ✅ 先删除依赖对象（视图、函数）
- ✅ CASCADE 选项
- ✅ 可选删除空 schema
- ✅ 手动代码块（备份数据）
- ✅ 详细回滚说明

**生成示例**:
```sql
-- ============================================================
-- Drop sys.users
-- File: 20251220000001_drop_sys_users.sql
-- Purpose:
--   - Drop User entity table
-- WARNING:
--   - This operation is DESTRUCTIVE and IRREVERSIBLE
--   - All data in the table will be lost
--   - Dependent objects (views, functions) may break
-- ============================================================

BEGIN;

SET LOCAL search_path = sys, public, extensions;

-- <<< block:custom_pre_drop
-- Add custom pre-drop SQL here (e.g., data backup)
-- >>> end:custom_pre_drop

-- Drop the table
DROP TABLE IF EXISTS sys.users CASCADE;

-- <<< block:custom_post_drop
-- Add custom post-drop SQL here
-- >>> end:custom_post_drop

COMMIT;

-- ============================================================
-- Rollback (manual recreation)
-- ============================================================
-- To rollback this drop, you would need to:
-- 1. Restore from backup
-- 2. Re-run the original CREATE TABLE migration
-- 3. Re-import data from backup
```

**YAML 配置**:
```yaml
- name: um_migration_drop_table
  description: "UM DROP TABLE Migration (WARNING: 破坏性操作)"
  template: "core_sql_migration/v1"
  template_file: "migration_drop_table.tera"
  target: "migrations/{{ migration_timestamp }}_drop_sys_users.sql"
  
  migration_description: "Drop users table (DESTRUCTIVE)"
```

---

### 4. `migration_constraints.tera`

**用途**: 生成约束迁移脚本（外键、唯一约束、检查约束、索引）

**特性**:
- ✅ 外键约束（FOREIGN KEY）
- ✅ 唯一约束（UNIQUE）
- ✅ 检查约束（CHECK）
- ✅ 索引（B-Tree、GIN、GiST、部分索引）
- ✅ ON DELETE/ON UPDATE 级联
- ✅ 手动代码块支持

**生成示例**:
```sql
-- ============================================================
-- Add/Modify Constraints for sys.users
-- File: 20251218000002_constraints_sys_users.sql
-- Purpose:
--   - Add or modify table constraints
-- ============================================================

BEGIN;

SET LOCAL search_path = sys, public, extensions;

-- ------------------------------------------------------------
-- Foreign Key Constraints
-- ------------------------------------------------------------
ALTER TABLE sys.users
  ADD CONSTRAINT fk_users_tenant
  FOREIGN KEY (tenant_id)
  REFERENCES sys.tenants(id)
  ON DELETE CASCADE
  ON UPDATE RESTRICT;

-- ------------------------------------------------------------
-- Unique Constraints
-- ------------------------------------------------------------
ALTER TABLE sys.users
  ADD CONSTRAINT uc_users_email_tenant
  UNIQUE (email, tenant_id);

-- ------------------------------------------------------------
-- Check Constraints
-- ------------------------------------------------------------
ALTER TABLE sys.users
  ADD CONSTRAINT chk_users_email_format
  CHECK (length(email) >= 3 AND length(email) <= 320);

-- ------------------------------------------------------------
-- Indexes (for performance)
-- ------------------------------------------------------------
CREATE INDEX IF NOT EXISTS idx_users_active_created
  ON sys.users USING btree (is_active, created_at)
  WHERE is_active = TRUE;

CREATE INDEX IF NOT EXISTS idx_users_roles
  ON sys.users USING gin (roles);

COMMIT;

-- ============================================================
-- Rollback (manual)
-- ============================================================
-- BEGIN;
-- ALTER TABLE sys.users DROP CONSTRAINT IF EXISTS fk_users_tenant;
-- ALTER TABLE sys.users DROP CONSTRAINT IF EXISTS uc_users_email_tenant;
-- ALTER TABLE sys.users DROP CONSTRAINT IF EXISTS chk_users_email_format;
-- DROP INDEX IF EXISTS sys.idx_users_active_created;
-- DROP INDEX IF EXISTS sys.idx_users_roles;
-- COMMIT;
```

**YAML 配置**:
```yaml
- name: um_migration_constraints
  description: "UM Constraints Migration"
  template: "core_sql_migration/v1"
  template_file: "migration_constraints.tera"
  target: "migrations/{{ migration_timestamp }}_constraints_sys_users.sql"
  
  migration_description: "Add foreign keys, unique constraints, and indexes"
  
  constraints:
    foreign_keys:
      - column: tenant_id
        target_schema: sys
        target_table: tenants
        target_column: id
        on_delete: CASCADE
        on_update: RESTRICT
        name: fk_users_tenant
    
    unique_constraints:
      - columns:
          - email
          - tenant_id
        name: uc_users_email_tenant
    
    check_constraints:
      - name: chk_users_email_format
        expression: "length(email) >= 3 AND length(email) <= 320"
    
    indexes:
      - columns:
          - is_active
          - created_at
        name: idx_users_active_created
        unique: false
        using: btree
        where: "is_active = TRUE"
      
      - columns:
          - roles
        name: idx_users_roles
        using: gin
```

---

## 🚀 使用方法

### 1. 配置 YAML

在 `teras/modules/{module}/tasks/migrations.yaml` 中配置任务。

### 2. 生成 Migration 文件

```bash
# 生成 CREATE TABLE Migration
tinyd tera gen --task um/migration_create_table

# 生成 ALTER TABLE Migration
tinyd tera gen --task um/migration_alter_table

# 查看差异
cat teras/generated/diff_report.json

# 应用到代码库
tinyd tera apply --task um/migration_create_table
```

### 3. 执行 Migration

```bash
# 应用迁移
tinyd migrate up

# 查看状态
tinyd migrate status

# 回滚迁移
tinyd migrate down
```

---

## 📊 SQL 类型映射

元数据丰富器自动映射 YAML 类型到 SQL 类型：

| YAML 类型 | SQL 类型 | 示例 |
|-----------|---------|------|
| `Uuid` | `UUID` | `'550e8400-e29b-41d4-a716-446655440000'` |
| `String` | `TEXT` | `'Hello World'` |
| `Email` | `CITEXT` | `'user@example.com'`（不区分大小写） |
| `i32` | `INTEGER` | `42` |
| `i64` | `BIGINT` | `9223372036854775807` |
| `bool` | `BOOLEAN` | `TRUE`/`FALSE` |
| `f32` | `REAL` | `3.14` |
| `f64` | `DOUBLE PRECISION` | `3.14159265359` |
| `Timestamp` | `TIMESTAMPTZ` | `'2025-01-01 12:00:00+00'` |
| `Vec<String>` | `TEXT[]` | `ARRAY['admin', 'user']` |
| `Json` | `JSONB` | `'{"key": "value"}'::jsonb` |

---

## 🛠️ 高级特性

### 1. Migration 依赖

```yaml
# um.yaml
entities:
  - name: User
    migration_dependencies:
      - "20251217000101_create_sys_schema.sql"
      - "20251001000001_create_extensions.sql"
```

生成的 Migration 会包含依赖注释。

### 2. Search Path 配置

```yaml
entities:
  - name: User
    search_path:
      - sys
      - public
      - extensions
```

生成：
```sql
SET LOCAL search_path = sys, public, extensions;
```

### 3. 索引类型

```yaml
constraints:
  indexes:
    - columns: [email]
      using: btree  # B-Tree索引（默认）
    
    - columns: [roles]
      using: gin    # GIN索引（数组/JSONB）
    
    - columns: [location]
      using: gist   # GiST索引（地理数据）
    
    - columns: [created_at]
      using: brin   # BRIN索引（时序数据）
```

### 4. 部分索引

```yaml
constraints:
  indexes:
    - columns: [is_active, created_at]
      where: "is_active = TRUE"  # 只索引活跃用户
```

### 5. 级联操作

```yaml
constraints:
  foreign_keys:
    - column: tenant_id
      target_table: tenants
      on_delete: CASCADE  # 删除租户时级联删除用户
      on_update: RESTRICT  # 限制更新租户ID
```

**级联选项**:
- `CASCADE` - 级联操作
- `RESTRICT` - 限制操作
- `SET NULL` - 设置为 NULL
- `SET DEFAULT` - 设置为默认值
- `NO ACTION` - 无操作（默认）

---

## ✅ 最佳实践

1. **命名规范**
   - Migration: `YYYYMMDDHHMMSS_description.sql`
   - 约束: `{type}_{table}_{column(s)}`
     - FK: `fk_users_tenant`
     - UK: `uc_users_email_tenant`
     - CK: `chk_users_email_format`
     - IX: `idx_users_is_active`

2. **Migration 顺序**
   - 先创建 schema
   - 再创建表
   - 最后添加约束和索引

3. **回滚准备**
   - 总是包含回滚脚本注释
   - 重要数据变更前备份
   - 测试环境先验证

4. **性能考虑**
   - 大表添加索引使用 `CONCURRENTLY`
   - 避免在事务中添加索引（除非必要）
   - 检查约束可能需要全表扫描

5. **安全性**
   - DROP 操作总是使用 `IF EXISTS`
   - 敏感操作使用手动代码块
   - 生产环境手动审查

---

## 🔄 工作流示例

```bash
# 1. 修改实体配置
vim teras/modules/um/um.yaml

# 2. 生成 Migration
tinyd tera gen --task um/migration_create_table

# 3. 查看差异
cat teras/generated/diff_report.json

# 4. 审查生成的 SQL
vim teras/generated/migrations/...sql

# 5. 应用到代码库
tinyd tera apply --task um/migration_create_table

# 6. 执行 Migration
tinyd migrate up

# 7. 验证表结构
psql -d tinydb -c "\d sys.users"

# 8. 提交更改
git add .
git commit -m "feat(migration): create sys.users table"
```

---

## ⚠️ 注意事项

1. **破坏性操作**
   - DROP TABLE/COLUMN 无法自动回滚
   - 总是先备份数据
   - 生产环境手动执行

2. **性能影响**
   - 添加NOT NULL约束需要全表扫描
   - 大表添加索引可能阻塞
   - 使用 `CONCURRENTLY` 减少锁定

3. **依赖关系**
   - 外键约束要求目标表存在
   - Migration 顺序很重要
   - 使用 `migration_dependencies` 声明依赖

---

## 📖 参考资料

- **PostgreSQL DDL**: [PostgreSQL Documentation](https://www.postgresql.org/docs/current/ddl.html)
- **Migration 最佳实践**: [Strong Migrations](https://github.com/ankane/strong_migrations)
- **索引优化**: [Use The Index, Luke](https://use-the-index-luke.com/)
- **SQLx Migrations**: [SQLx CLI](https://github.com/launchbadge/sqlx/tree/main/sqlx-cli)

---

**模板版本**: v1  
**最后更新**: 2025-01-02  
**维护者**: TinyDev Team
