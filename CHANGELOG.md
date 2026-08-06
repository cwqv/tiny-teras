# Changelog

## [1.0.0] - 2026-08

### Added
- Extracted from `tiny` monorepo `dev_infra/teras/`.
- 67 template files (core_rust / core_flutter / core_proto / workspace / erp).
- 28 module configs (um / erp / workspace-task / workspace-memo / examples).
- `registry.yaml` with task groups (backend/frontend/all/core/proto/migrations/erp).

### Notes
- `generated/` and `backups/` are build artifacts, gitignored.
- `codebase.rust_path/flutter_path` in registry.yaml reference the legacy
  `core_rust`/`core_flutter` layout; consumers override the target repo root
  via `TINY_PROJECT_ROOT` (see README.md).
