# Access Control

DIGIT uses a three-part authorization system: **Roles** define who users are, **Actions** define what API endpoints exist, and **Role-Actions** map which roles can call which APIs. Every API request passes through `egov-access-control` which checks this permission matrix.

## Files

| File | Schema Code | Records | Purpose |
|------|------------|---------|---------|
| `mdms/ACCESSCONTROL-ROLES/roles.json` | `ACCESSCONTROL-ROLES.roles` | ~23 roles per tenant | Role definitions |
| `mdms/ACCESSCONTROL-ACTIONS-TEST/actions-test.json` | `ACCESSCONTROL-ACTIONS-TEST.actions-test` | 196 actions | API endpoint registry |
| `mdms/ACCESSCONTROL-ROLEACTIONS/roleactions.json` | `ACCESSCONTROL-ROLEACTIONS.roleactions` | 341 mappings | Permission matrix |

---

## ACCESSCONTROL-ROLES.roles

Defines every role in the system. Each role is registered per tenant (root level). City tenants inherit from their root.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "code": "GRO",
    "name": "Complaint Assessor",
    "description": "One who will assess & assign complaints"
  }
}
```

### PGR Roles (Core)

These roles are directly used in the PGR workflow state machine:

| Role Code | Name | Purpose in PGR |
|-----------|------|---------------|
| `CITIZEN` | Citizen | Files complaints, reopens, rates resolved complaints |
| `CSR` | Customer Service Rep | Files complaints on behalf of citizens |
| `GRO` | Grievance Routing Officer | Assesses complaints, assigns to field workers, can reject |
| `DGRO` | Department GRO | Same as GRO but scoped to a department |
| `PGR_LME` | Last Mile Employee | Resolves complaints in the field, can reassign |
| `PGR_VIEWER` | PGR Viewer | Read-only access plus can assign/reject (supervisory) |
| `SUPERVISOR` | Supervisor | Handles auto-escalated complaints |
| `AUTO_ESCALATE` | Auto Escalation | System role for automatic escalation timers |

### Platform Roles (Infrastructure)

| Role Code | Name | Purpose |
|-----------|------|---------|
| `EMPLOYEE` | Employee | Base role for all government employees. Required alongside specific PGR roles. |
| `SUPERUSER` | Super User | Full system administrator access |
| `MDMS_ADMIN` | MDMS Admin | Can create/edit MDMS schemas and data |
| `LOC_ADMIN` | Localization Admin | Can edit localization messages |
| `WORKFLOW_ADMIN` | Workflow Admin | Can create/edit workflow definitions |
| `HRMS_ADMIN` | HRMS Admin | Can manage employee records |
| `INTERNAL_MICROSERVICE_ROLE` | Internal Microservice | Service-to-service calls, gets full PII access |
| `REINDEXING_ROLE` | Reindexing | Used for Elasticsearch reindexing operations |
| `SYSTEM` | System User | Internal system operations |
| `ANONYMOUS` | Anonymous | Unauthenticated access (limited to public APIs) |
| `COMMON_EMPLOYEE` | Basic Employee | Basic employee permissions |
| `QA_AUTOMATION` | QA Automation | Automated testing role |
| `TICKET_REPORT_VIEWER` | Report Viewer | Can view PGR analytics reports |
| `CFC` | CFC | Citizen Facilitation Center operator |

### How Other Configs Use This

- **Workflow**: Transition rules reference role codes (e.g., only `GRO` can `ASSIGN`).
- **Role-Actions**: Maps these roles to specific API permissions.
- **DataSecurity**: `SecurityPolicy.roleBasedDecryptionPolicy` references these role codes to control PII visibility.
- **HRMS**: Employee creation assigns roles from this list. Never assign `CITIZEN` or `CSR` to an employee via HRMS.

---

## ACCESSCONTROL-ACTIONS-TEST.actions-test

Registers every API endpoint as a numbered "action". Kong and `egov-access-control` use this registry to validate requests.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "id": 2149,
    "url": "/pgr-services/v2/request/_create",
    "enabled": true,
    "displayName": "Create PGR Service Request",
    "orderNumber": 0,
    "serviceCode": "pgr-services",
    "parentModule": ""
  }
}
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Numeric ID. Must be unique. Used in role-action mappings. |
| `url` | Yes | API path pattern that Kong matches against incoming requests. |
| `enabled` | Yes | Whether this action is active. Disabled actions are ignored. |
| `displayName` | Yes | Human-readable name (not shown in UI, used for admin reference). |
| `serviceCode` | Yes | Groups actions by service (e.g., `pgr-services`, `egov-hrms`). |

### Key PGR Actions

| ID | URL | Description |
|----|-----|-------------|
| 2149 | `/pgr-services/v2/request/_create` | File a complaint |
| 2150 | `/pgr-services/v2/request/_search` | Search complaints |
| 2151 | `/pgr-services/v2/request/_update` | Update complaint (assign, resolve, etc.) |
| 2152 | `/pgr-services/v2/request/_count` | Count complaints |

### Legacy Actions

The actions list includes endpoints for modules not used in CCRS (Property Tax, Trade License, Water & Sewerage, Fire NOC, Building Permits). These are inherited from the full DIGIT platform and do not cause issues -- they are simply unused permission entries.

---

## ACCESSCONTROL-ROLEACTIONS.roleactions

The permission matrix. Each record maps one role to one action. If a mapping does not exist, the role cannot call that API.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "actionid": 2149,
    "rolecode": "CITIZEN",
    "tenantId": "ke",
    "actioncode": ""
  }
}
```

This record means: "Users with `CITIZEN` role can call action ID 2149 (`/pgr-services/v2/request/_create`)."

### Permission Distribution

| Role | Actions Allowed | Key Permissions |
|------|----------------|-----------------|
| `CITIZEN` | 37 | File complaints, search own complaints, user profile, localization |
| `GRO` | 37 | Assign complaints, search all complaints, HRMS search, workflow |
| `CSR` | 28 | File on behalf of citizen, search complaints |
| `PGR_LME` | 22 | Resolve complaints, reassign, search |
| `EMPLOYEE` | 29 | Basic employee access, search, localization |
| `SUPERUSER` | 74 | Full admin access to all APIs |
| `MDMS_ADMIN` | 87 | All MDMS schema and data management APIs |
| `PGR_VIEWER` | 7 | Read-only PGR access |
| `HRMS_ADMIN` | 7 | Employee management APIs |
| `ANONYMOUS` | ~4 | Public APIs only (localization, filestore) |

### How the Authorization Flow Works

```
1. User makes API call → Kong gateway
2. Kong extracts auth token → calls egov-user to get user roles
3. Kong calls egov-access-control with (roles, requested URL)
4. egov-access-control:
   a. Finds the action ID matching the URL (from actions-test)
   b. Checks if any of the user's roles have a roleaction mapping for that action ID
   c. Returns allowed/denied
5. Kong forwards to backend service or returns 403
```

### Dependency

**Required by**: Every API call. Without role-actions, all requests return 403.

**Depends on**: `ACCESSCONTROL-ROLES.roles` (role codes must exist), `ACCESSCONTROL-ACTIONS-TEST.actions-test` (action IDs must exist).
