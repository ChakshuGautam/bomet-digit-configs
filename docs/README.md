# DIGIT Configuration Reference

This directory documents every MDMS config, localization module, and workflow definition used by the Bomet County DIGIT deployment. Each document explains what the config does, what fields it contains, and how it connects to other configs.

## How DIGIT Configuration Works

DIGIT externalizes all behavior to configuration. The platform has zero hardcoded business logic for complaint types, permissions, workflows, or UI labels. Everything is stored in **MDMS v2** (Master Data Management System) as JSON records, organized by `module.master` schema codes.

Every MDMS record has a `tenantId` that scopes it to a specific government entity. Records at the root tenant (e.g., `ke`) are inherited by all child tenants (e.g., `ke.bomet`). City-level records override or supplement root-level ones.

## Configuration Dependency Chain

Configs must be seeded in this order. Later levels depend on earlier ones.

```
Level 0: MDMS Schemas
  Must exist before any data records can be created.

Level 1: Core Platform Data (independent of each other)
  +-- Access Control Roles .............. docs/access-control.md
  +-- Data Security Policies ............ docs/data-security.md
  +-- Common Masters .................... docs/common-masters.md
  +-- HRMS Masters ...................... docs/hrms.md
  +-- Tenant Config ..................... docs/tenant.md
  +-- Localization ...................... docs/localization.md

Level 2: Permission Matrix
  +-- Actions (API endpoint registry)
  +-- Role-Actions (which roles call which APIs)
  Both documented in .................... docs/access-control.md

Level 3: Users
  Depends on: Roles existing in Level 1

Level 4: Employees
  Depends on: Users + Departments + Designations + HRMS masters

Level 5: Workflow
  Depends on: Roles (CITIZEN, GRO, PGR_LME used in transitions)
  Documented in ......................... docs/workflow.md

Level 6: PGR Module Data
  +-- ServiceDefs (complaint types) ..... docs/pgr.md
  +-- UIConstants ....................... docs/pgr.md
  +-- Inbox Configuration .............. docs/inbox.md
  +-- Boundaries ....................... docs/boundary.md
```

## Cross-Reference: Which Service Reads Which Config?

| DIGIT Service | Configs Read | Purpose |
|--------------|-------------|---------|
| **egov-access-control** | `ACCESSCONTROL-ROLES.roles`, `ACCESSCONTROL-ACTIONS-TEST.actions-test`, `ACCESSCONTROL-ROLEACTIONS.roleactions` | API-level authorization |
| **egov-enc-service** | `DataSecurity.SecurityPolicy`, `DataSecurity.EncryptionPolicy`, `DataSecurity.MaskingPatterns`, `DataSecurity.DecryptionABAC` | PII encryption/masking |
| **egov-workflow-v2** | `Workflow.BusinessService`, `Workflow.AutoEscalation`, `Workflow.AutoEscalationStatesToIgnore`, `Workflow.BusinessServiceConfig` | Complaint lifecycle state machine |
| **pgr-services** | `RAINMAKER-PGR.ServiceDefs`, `RAINMAKER-PGR.UIConstants` | Complaint types and SLA |
| **egov-hrms** | `egov-hrms.EmployeeType`, `egov-hrms.EmployeeStatus`, `egov-hrms.DeactivationReason`, etc. | Employee management |
| **egov-user** | `common-masters.UserValidation`, `common-masters.GenderType` | User registration |
| **egov-idgen** | `common-masters.IdFormat` | Complaint ID generation |
| **DIGIT UI** | `common-masters.StateInfo`, `common-masters.uiHomePage`, `tenant.citymodule`, `tenant.tenants`, localization messages | UI rendering |
| **inbox-v2** | `INBOX.InboxQueryConfiguration` | Employee complaint inbox |
| **boundary-service** | `egov-location.TenantBoundary` | HRMS jurisdiction hierarchy |

## Index

| Document | Configs Covered | Records |
|----------|----------------|---------|
| [tenant.md](tenant.md) | `tenant.tenants`, `tenant.citymodule` | Tenant hierarchy and module activation |
| [access-control.md](access-control.md) | `ACCESSCONTROL-ROLES.roles`, `ACCESSCONTROL-ACTIONS-TEST.actions-test`, `ACCESSCONTROL-ROLEACTIONS.roleactions` | 23 roles, 196 actions, 341 role-action mappings |
| [data-security.md](data-security.md) | `DataSecurity.SecurityPolicy`, `EncryptionPolicy`, `DecryptionABAC`, `MaskingPatterns` | PII protection for 32 data models |
| [common-masters.md](common-masters.md) | `common-masters.Department`, `Designation`, `GenderType`, `IdFormat`, `StateInfo`, `uiHomePage`, `wfSlaConfig`, `CronJobAPIConfig`, `UserValidation` | Shared lookup tables |
| [pgr.md](pgr.md) | `RAINMAKER-PGR.ServiceDefs`, `RAINMAKER-PGR.UIConstants` | 47 Bomet complaint types |
| [hrms.md](hrms.md) | `egov-hrms.EmployeeType`, `EmployeeStatus`, `DeactivationReason`, `Degree`, `Specalization`, `EmploymentTest` | Employee metadata |
| [workflow.md](workflow.md) | `Workflow.BusinessService`, `BusinessServiceConfig`, `AutoEscalation`, `AutoEscalationStatesToIgnore`, `BusinessServiceMasterConfig` + `workflows/business-services.json` | PGR state machine |
| [inbox.md](inbox.md) | `INBOX.InboxQueryConfiguration` | Elasticsearch query config for employee inbox |
| [boundary.md](boundary.md) | `egov-location.TenantBoundary`, `CRS-ADMIN-CONSOLE.adminSchema` | Bomet's 5 subcounties, 25 wards |
| [localization.md](localization.md) | All `localization/` files | 51,758 messages across 3 locales |
