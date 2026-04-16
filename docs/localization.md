# Localization

Every label, button, error message, status name, and complaint type in the DIGIT UI comes from the localization service. Without localization messages, the UI renders raw message codes like `CS_COMMON_COMPLAINT_SUBMITTED` instead of "Complaint Submitted Successfully".

## Directory Structure

```
localization/
+-- en_IN/                    # English (India)
|   +-- rainmaker-common.json
|   +-- rainmaker-pgr.json
|   +-- rainmaker-hr.json
|   +-- rainmaker-workbench.json
|   +-- rainmaker-pg.json
|   +-- rainmaker-hrms.json
|   +-- egov-user.json
|   +-- egov-hrms.json
|
+-- hi_IN/                    # Hindi (India)
|   +-- rainmaker-common.json
|   +-- rainmaker-pgr.json
|   +-- rainmaker-hr.json
|   +-- rainmaker-workbench.json
|   +-- egov-user.json
|   +-- egov-hrms.json
|
+-- default/                  # Default/fallback locale
    +-- rainmaker-common.json
    +-- rainmaker-pgr.json
    +-- rainmaker-hr.json
    +-- rainmaker-workbench.json
    +-- egov-user.json
    +-- egov-hrms.json
```

## Message Counts

| Module | en_IN | hi_IN | default | Total |
|--------|-------|-------|---------|-------|
| `rainmaker-common` | ~2,400 | ~7,000 | ~1,700 | ~11,100 |
| `rainmaker-pgr` | ~2,000 | ~1,100 | ~1,400 | ~4,500 |
| `rainmaker-workbench` | ~800 | ~1,600 | ~800 | ~3,200 |
| `rainmaker-hr` | ~400 | ~400 | ~400 | ~1,200 |
| `rainmaker-pg` | ~30 | - | - | ~30 |
| `rainmaker-hrms` | ~4 | - | - | ~4 |
| `egov-user` | ~7 | ~5 | ~4 | ~16 |
| `egov-hrms` | ~3 | ~2 | ~2 | ~7 |

**Total**: ~51,758 messages across all locales.

---

## Modules

### rainmaker-common

Shared UI labels used across all modules. The largest module.

**Key patterns**:
| Key Pattern | Example | Purpose |
|------------|---------|---------|
| `CS_COMMON_*` | `CS_COMMON_SUBMIT` → "Submit" | Common actions |
| `CORE_COMMON_*` | `CORE_COMMON_SEARCH` → "Search" | Core platform labels |
| `COMMON_MASTERS_DEPARTMENT_*` | `COMMON_MASTERS_DEPARTMENT_DEPT_3` → "Health & Sanitation" | Department display names |
| `COMMON_MASTERS_DESIGNATION_*` | `COMMON_MASTERS_DESIGNATION_DESIG_1002` → "Field Worker" | Designation display names |
| `CS_HEADER_*` | `CS_HEADER_COMPLAINT_TRACKING` → "Complaint Tracking" | Page headers |
| `ERR_*` | `ERR_FIELD_REQUIRED` → "This field is required" | Validation errors |
| `EGOV_LOCATION_BOUNDARYTYPE_*` | `EGOV_LOCATION_BOUNDARYTYPE_WARD` → "Ward" | Boundary type labels |
| `ADMIN_*` | `ADMIN_SUBCOUNTY` → "Sub County" | Boundary hierarchy labels |
| `<BOUNDARY_CODE>` | `BOMET_BOMET_CENTRAL` → "Central" | Boundary name display |

### rainmaker-pgr

PGR-specific labels for complaint filing, tracking, and management.

**Key patterns**:
| Key Pattern | Example | Purpose |
|------------|---------|---------|
| `SERVICEDEFS.*` | `SERVICEDEFS.BurningOfGarbage` → "Burning of garbage" | Complaint type names |
| `CS_COMPLAINT_*` | `CS_COMPLAINT_DETAILS` → "Complaint Details" | Complaint UI labels |
| `CS_ADDCOMPLAINT_*` | `CS_ADDCOMPLAINT_PLACEHOLDER_DESCRIPTION` → "Enter complaint description" | Filing form labels |
| `WF_PGR_*` | `WF_PGR_PENDINGFORASSIGNMENT` → "Pending for Assignment" | Workflow state labels |
| `CATEGORY_*` | `CATEGORY_Garbage` → "Garbage" | Service definition categories |
| `CS_HEADER_PGR_*` | `CS_HEADER_PGR_COMPLAINTS` → "Complaints" | PGR page headers |

### rainmaker-hr

HRMS module labels for employee management.

**Key patterns**:
| Key Pattern | Example | Purpose |
|------------|---------|---------|
| `HR_EMPLOYEE_*` | `HR_EMPLOYEE_CREATE` → "Create Employee" | Employee management labels |
| `HR_COMMON_*` | `HR_COMMON_DEPARTMENT` → "Department" | Common HRMS form labels |

### rainmaker-workbench

Admin workbench labels for MDMS data management.

### egov-user

User authentication labels (login, registration, OTP).

### egov-hrms

HRMS core system messages.

---

## How Localization Works

### Message Structure

Each message has:
```json
{
  "code": "CS_COMMON_SUBMIT",
  "message": "Submit",
  "module": "rainmaker-common",
  "locale": "en_IN"
}
```

### Resolution Order

When the UI needs to display a label:

1. Look up the message code in the user's selected locale (e.g., `en_IN`)
2. If not found, fall back to `default` locale
3. If still not found, display the raw code (e.g., `CS_COMMON_SUBMIT`)

### Which Modules Are Loaded

The `common-masters.StateInfo` config has a `localizationModules` array that lists which modules to load. Only listed modules are fetched from the localization service.

### Adding New Localizations

New localizations are added via the localization API:
```
POST /localization/messages/v1/_upsert
```

The upsert operation creates new messages or updates existing ones. It is idempotent.

---

## Connection to Other Configs

| Config | Localization Keys Generated |
|--------|----------------------------|
| `RAINMAKER-PGR.ServiceDefs` | `SERVICEDEFS.<serviceCode>` for each complaint type |
| `common-masters.Department` | `COMMON_MASTERS_DEPARTMENT_<code>` for each department |
| `common-masters.Designation` | `COMMON_MASTERS_DESIGNATION_<code>` for each designation |
| `egov-location.TenantBoundary` | `<BOUNDARY_CODE>` for each boundary, `EGOV_LOCATION_BOUNDARYTYPE_<LEVEL>` for types |
| `ACCESSCONTROL-ROLES.roles` | Role labels in workflow action buttons |
| Workflow states | `WF_PGR_<STATE>` for each workflow state |

### Without Localization

If localization messages are missing:
- Complaint types show as `SERVICEDEFS.BurningOfGarbage`
- Workflow states show as `WF_PGR_PENDINGFORASSIGNMENT`
- Boundary names show as `BOMET_BOMET_CENTRAL_CHESOEN`
- Department names show as `COMMON_MASTERS_DEPARTMENT_DEPT_3`
- All buttons show their code keys instead of labels

The platform is technically functional but completely unusable for end users.

---

## Dependency Summary

| Config | Depends On | Required By |
|--------|-----------|-------------|
| Localization messages | `common-masters.StateInfo` (module loading list) | Every DIGIT UI screen |
| Message codes | All MDMS configs (codes must match) | UI label rendering |
