# Common Masters

Shared lookup tables used across all DIGIT modules. These define departments, designations, gender types, ID generation formats, UI branding, and other platform-wide configuration.

## Files

| File | Schema Code | Purpose |
|------|------------|---------|
| `mdms/common-masters/Department.json` | `common-masters.Department` | Government departments for complaint routing |
| `mdms/common-masters/Designation.json` | `common-masters.Designation` | Employee job titles |
| `mdms/common-masters/GenderType.json` | `common-masters.GenderType` | Gender dropdown values |
| `mdms/common-masters/IdFormat.json` | `common-masters.IdFormat` | Complaint/receipt ID patterns |
| `mdms/common-masters/StateInfo.json` | `common-masters.StateInfo` | UI branding and locale config |
| `mdms/common-masters/uiHomePage.json` | `common-masters.uiHomePage` | Citizen home page layout |
| `mdms/common-masters/wfSlaConfig.json` | `common-masters.wfSlaConfig` | SLA traffic light colors |
| `mdms/common-masters/CronJobAPIConfig.json` | `common-masters.CronJobAPIConfig` | Scheduled job configuration |
| `mdms/common-masters/UserValidation.json` | `common-masters.UserValidation` | Mobile number validation rules |

---

## common-masters.Department

Government departments that handle complaints. Each complaint type (`RAINMAKER-PGR.ServiceDefs`) references a department code.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "code": "DEPT_3",
    "name": "Health & Sanitation",
    "active": true
  }
}
```

### Departments

| Code | Name | PGR Complaint Types Assigned |
|------|------|----------------------------|
| `DEPT_1` | Street Lights | Streetlight complaints |
| `DEPT_2` | Building & Roads | Road/footpath damage |
| `DEPT_3` | Health & Sanitation | Garbage, dead animals, public toilets |
| `DEPT_4` | Operation & Maintenance | Water/sewerage, construction debris |
| `DEPT_5` | Horticulture | Tree cutting/trimming |
| `DEPT_6` | Building Branch | Illegal constructions, parking |
| `DEPT_7` | Citizen Service Desk | General citizen queries |
| `DEPT_8` | Complaint Cell | Complaint management |
| `DEPT_9` | Executive Branch | Executive-level issues |
| `DEPT_10` | Others | Uncategorized |
| `DEPT_13` | Tax Branch | Tax-related |
| `DEPT_25` | Accounts Branch | Financial |
| `DEPT_35` | Works Branch | Public works |
| `DEPT_36` | HealthServices | Bomet health-specific complaints |

### Connection to Other Configs

- **ServiceDefs**: Each complaint type has a `department` field referencing a code from this list (e.g., `"department": "DEPT_36"`).
- **HRMS**: Employees are assigned to departments. A GRO in `DEPT_36` handles health complaints.
- **Localization**: UI labels for departments come from localization messages with key pattern `COMMON_MASTERS_DEPARTMENT_<CODE>`.

---

## common-masters.Designation

Employee job titles. Required when creating employees via HRMS.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "code": "DESIG_1002",
    "name": "Field Worker",
    "department": ["DEPT_36"]
  }
}
```

### Key Designations

| Code | Name | Department Scope |
|------|------|-----------------|
| `AO` | Accounts Officer | - |
| `COMM` | Commissioner | - |
| `DESIG_01` | Assistant Engineer | - |
| `DESIG_02` | Junior Engineer | - |
| `DESIG_04` | Sanitary Inspector | - |
| `DESIG_05` | Tax Inspector | - |
| `DESIG_1002` | Field Worker | `DEPT_36` (HealthServices) |
| `DESIG_1003` | Nursing Officer | `DEPT_36` (HealthServices) |
| `DESIG_1004` | Administrator | `DEPT_36` (HealthServices) |

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `code` | Yes | Unique identifier. |
| `name` | Yes | Display name in HRMS UI. |
| `department` | No | Array of department codes this designation applies to. If set, the UI only shows this designation for employees in those departments. |

### Connection to Other Configs

- **HRMS**: Employee creation requires a valid designation code.
- **Localization**: Labels come from `COMMON_MASTERS_DESIGNATION_<CODE>`.

---

## common-masters.GenderType

Gender options for user registration and employee creation forms.

### Values

| Code | Active | Description |
|------|--------|-------------|
| `MALE` | true | Male |
| `FEMALE` | true | Female |
| `TRANSGENDER` | true | Transgender |
| `OTHERS` | false | Others (disabled) |

### Connection to Other Configs

- **egov-user**: User registration form populates gender dropdown from this.
- **HRMS**: Employee creation form uses this for the gender field.

---

## common-masters.IdFormat

Defines patterns for auto-generating IDs (complaint numbers, receipt numbers, etc.) via `egov-idgen`.

### PGR-Relevant Format

```json
{
  "idname": "pgr.servicerequestid",
  "format": "PG-PGR-[cy:yyyy-MM-dd]-[SEQ_EG_PGR_ID]"
}
```

This generates complaint IDs like: `PG-PGR-2026-04-16-000042`

### Format Tokens

| Token | Description |
|-------|-------------|
| `[CITY.CODE]` | City code from `tenant.tenants.city.code` |
| `[cy:yyyy-MM-dd]` | Current date in the specified format |
| `[fy:yyyy-yy]` | Financial year (e.g., `2026-27`) |
| `[SEQ_EG_PGR_ID]` | Auto-incrementing sequence from `egov-idgen` |

### Connection to Other Configs

- **pgr-services**: Calls `egov-idgen` when creating a complaint. `egov-idgen` reads this config to generate the ID.
- **tenant.tenants**: The `[CITY.CODE]` token is resolved from `tenant.tenants.city.code`.

---

## common-masters.StateInfo

UI branding, locale settings, and module list for the DIGIT frontend. One record per tenant.

### Record Structure (simplified)

```json
{
  "code": "ke.bomet",
  "name": "Bomet County",
  "logoUrl": "https://...bomet-county-logo.jpg",
  "bannerUrl": "https://...banner.jpg",
  "statelogo": "https://...logo.jpg",
  "languages": [
    { "label": "ENGLISH", "value": "en_IN" }
  ],
  "hasLocalisation": true,
  "localizationModules": [
    { "label": "rainmaker-common", "value": "rainmaker-common" },
    { "label": "rainmaker-pgr", "value": "rainmaker-pgr" },
    { "label": "rainmaker-hr", "value": "rainmaker-hr" }
  ],
  "defaultUrl": {
    "citizen": "/user/register",
    "employee": "/user/login"
  }
}
```

### Key Fields

| Field | Description |
|-------|-------------|
| `logoUrl` | Header logo displayed on every page |
| `bannerUrl` | Background banner on the login page |
| `languages` | Available language selector options. `value` must match a locale folder in `localization/`. |
| `localizationModules` | Which localization modules to load at startup. The UI only fetches messages from these modules. |
| `defaultUrl.citizen` | Landing page URL for citizen users |
| `defaultUrl.employee` | Landing page URL for employee users |

### Connection to Other Configs

- **Localization**: The `localizationModules` list determines which localization files are loaded. If `rainmaker-pgr` is missing, PGR labels won't render.
- **DIGIT UI**: Reads this on startup for branding, language options, and routing.

---

## common-masters.uiHomePage

Configures the citizen-facing home page layout: banner images, service cards, and navigation links.

### Key Sections

| Section | Description |
|---------|-------------|
| `appBannerMobile` | Banner image for mobile view |
| `appBannerDesktop` | Banner image for desktop view |
| `citizenServicesCard` | Service cards shown on home page (PGR, Property Tax, etc.) |
| `redirectURL` | Default redirect after login (`"all-services"`) |

### Connection to Other Configs

- **DIGIT UI**: Renders the citizen home page from this config.
- **tenant.citymodule**: Services listed here should also be active in `citymodule`.

---

## common-masters.wfSlaConfig

SLA (Service Level Agreement) traffic light colors for the employee inbox. Complaints are color-coded based on how much of the SLA period has elapsed.

### Record Structure

```json
{
  "slotPercentage": 33,
  "positiveSlabColor": "#4CAF50",
  "middleSlabColor": "#EEA73A",
  "negativeSlabColor": "#F44336"
}
```

| Field | Value | Meaning |
|-------|-------|---------|
| `slotPercentage` | 33 | Each color zone covers 33% of the SLA period |
| `positiveSlabColor` | `#4CAF50` (green) | Within first 33% of SLA time |
| `middleSlabColor` | `#EEA73A` (amber) | Between 33-66% of SLA time |
| `negativeSlabColor` | `#F44336` (red) | Beyond 66% of SLA time (overdue risk) |

### Connection to Other Configs

- **DIGIT UI Inbox**: Colors complaint rows in the inbox based on SLA elapsed time.
- **Workflow**: SLA duration comes from the workflow definition (e.g., 5 days for PGR).

---

## common-masters.CronJobAPIConfig

Configuration for scheduled cron jobs (e.g., auto-escalation timers). Not commonly modified.

---

## common-masters.UserValidation

Mobile number validation rules for user registration.

### Connection to Other Configs

- **egov-user**: Validates mobile numbers against these rules during user creation.
- **DIGIT UI**: Client-side validation on registration/login forms.

---

## Dependency Summary

| Config | Depends On | Required By |
|--------|-----------|-------------|
| `Department` | Nothing | `RAINMAKER-PGR.ServiceDefs`, HRMS employee creation |
| `Designation` | `Department` (optional scope) | HRMS employee creation |
| `GenderType` | Nothing | User/employee creation forms |
| `IdFormat` | `tenant.tenants` (for `[CITY.CODE]`) | `egov-idgen` → `pgr-services` |
| `StateInfo` | Nothing | DIGIT UI startup |
| `uiHomePage` | Nothing | DIGIT UI citizen landing page |
| `wfSlaConfig` | Nothing | DIGIT UI inbox |
