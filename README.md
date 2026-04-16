# Bomet DIGIT Configs

Configuration dump from the Bomet County DIGIT deployment (`egov-bomet` / `bometfeedbackhub.digit.org`).

**Dump date**: 2026-04-16

## Contents

### `mdms/` — MDMS v2 Data (33 schemas, 7,846 records)
Master Data Management records organized by module:
- `tenant/` — Tenant definitions and citymodule config
- `ACCESSCONTROL-*/` — Roles, role-actions, actions
- `common-masters/` — Departments, designations, gender types, StateInfo
- `RAINMAKER-PGR/` — PGR service definitions, UI constants
- `DataSecurity/` — Encryption policies, masking patterns, security policies
- `egov-hrms/` — Employee types, statuses, deactivation reasons
- `egov-location/` — TenantBoundary (HRMS jurisdiction hierarchy)
- `Workflow/` — Business service configs, auto-escalation rules
- `INBOX/` — Inbox query configurations

### `mdms-schemas/` — MDMS Schema Definitions (346 schemas)
JSON Schema definitions for all MDMS masters.

### `localization/` — Localization Messages (51,758 messages)
i18n messages organized by locale (`en_IN`, `hi_IN`, `default`) and module.

### `workflows/` — Workflow Business Services
PGR and other workflow state machine definitions.

### `egov-db-dump.sql.gz` — Full Database Dump (2.8 MB compressed)
Complete PostgreSQL dump (`pg_dump`) of the `egov` database including all tables:
MDMS, localization, users, PGR complaints, workflows, boundaries, etc.

```bash
# Restore to a local postgres:
gunzip -c egov-db-dump.sql.gz | psql -U egov -d egov
```

## Source Server
- **Host**: egov-bomet (10.0.0.2 / 62.238.3.196)
- **Domain**: bometfeedbackhub.digit.org
- **Stack**: DIGIT Docker Compose (~25 containers)
- **Tenant**: `ke.bomet` (root: `ke`)
