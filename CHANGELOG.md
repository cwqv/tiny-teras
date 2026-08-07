# Changelog

## [Unreleased]

### Removed
- Removed `modules/erp/` (8 entity YAMLs + README) and 6 `templates/erp_*/` families
  (`erp_computed_fields`, `erp_domain_event`, `erp_header_line_repo`,
  `erp_state_machine`, `erp_flutter_list_page`, `erp_flutter_form_page`).
  These were extracted from the monorepo but never wired into the registry:
  the ERP modules had no `tasks/` directory, the `erp_*` templates had zero
  references, and the registry's `erp`/`erp_backend`/`erp_frontend`/`product`
  groups pointed at non-existent task lists. ERP codegen will be reintroduced
  as a follow-up once the task-wiring design is finalized.

### Changed
- `registry.yaml`: removed the dead `product`, `erp`, `erp_backend`, and
  `erp_frontend` task groups (they referenced missing modules/tasks).

### Fixed
- `.gitignore`: ignore `.reasonix/`, `reasonix.toml`, `.workbuddy/` tooling-private state.

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
