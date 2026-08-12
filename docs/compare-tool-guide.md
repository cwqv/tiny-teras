# tinyd tera compare 工具用户手册

> 日期：2026-08-12
> 状态：随 tera-erp-codegen-refactor G8 固化

## 用途

量化 tera 模板的代码覆盖率：对比 `generated/` 产物与仓库内手写源码，计算 `生成行数 / 手写行数`，并列出缺失项。

## 用法

```bash
# 按模块对比全部文件
tinyd tera compare --module account

# 仅对比单个文件族（entity / dto / command / query / repository）
tinyd tera compare --module account --file repository

# 输出 markdown 报告
tinyd tera compare --module account --output tera-compare-account.md

# JSON 输出（CI 消费）
tinyd tera compare --module account --json

# 覆盖率回归检测（低于阈值非零退出）
tinyd tera compare --module account --min-coverage 35
```

## 输出示例

```
📄 postgres_account_repository.rs
  生成: 202 行
  手工: 305 行
  覆盖率: 66.2%
```

## JSON 结构

```json
{
  "files": [
    { "name": "postgres_account_repository.rs", "generated_lines": 202,
      "handwritten_lines": 305, "coverage": 66.2,
      "missing_items": ["find_by_code", "list", "count"] }
  ]
}
```

## CI 集成

```yaml
# 示例：repo 模板覆盖率回归检测
- run: tinyd tera compare --module account --file repository --min-coverage 35
```

- 覆盖率 < 阈值 → 非零退出 → CI 失败（防止模板回归）。
- 首次接入建议先用宽松阈值（如 30%），稳定后收紧。

## 注意事项

- compare 依赖 `generated/` 产物，先跑 `tinyd tera gen --group <module>`。
- 覆盖率是**骨架生成度**指标，不代表业务完整性——复杂方法（JOIN/聚合/动态 SQL）永远手写，不计入缺失。
