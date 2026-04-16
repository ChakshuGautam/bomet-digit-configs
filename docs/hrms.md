# HRMS Masters (Human Resource Management System)

HRMS master data defines the dropdown options and validation rules for employee management. These are lookup tables -- the actual employee records are stored in the `egov-hrms` database, not in MDMS.

## Files

| File | Schema Code | Purpose |
|------|------------|---------|
| `mdms/egov-hrms/EmployeeType.json` | `egov-hrms.EmployeeType` | Employment type options |
| `mdms/egov-hrms/EmployeeStatus.json` | `egov-hrms.EmployeeStatus` | Employee status lifecycle |
| `mdms/egov-hrms/DeactivationReason.json` | `egov-hrms.DeactivationReason` | Reasons for deactivating an employee |
| `mdms/egov-hrms/Degree.json` | `egov-hrms.Degree` | Educational qualifications |
| `mdms/egov-hrms/Specalization.json` | `egov-hrms.Specalization` | Professional specializations |
| `mdms/egov-hrms/EmploymentTest.json` | `egov-hrms.EmploymentTest` | Employment test types |

---

## egov-hrms.EmployeeType

Employment classification. Required field when creating an employee.

### Values

| Code | Description |
|------|-------------|
| `PERMANENT` | Permanent government employee |
| `TEMPORARY` | Temporary/short-term employee |
| `DAILYWAGES` | Daily wage worker |
| `CONTRACT` | Contract-based employee |
| `DEPUTATION` | Employee on deputation from another department |

### Connection to Other Configs

- **HRMS**: The "Create Employee" form has an "Employment Type" dropdown populated from this.

---

## egov-hrms.EmployeeStatus

Employee lifecycle status. Controls whether an employee can log in and be assigned complaints.

### Values

| Code | Description | Can Login? | Can Receive Complaints? |
|------|-------------|-----------|----------------------|
| `EMPLOYED` | Active employee | Yes | Yes |
| `RETIRED` | Retired | No | No |
| `RESIGNED` | Voluntarily left | No | No |
| `TERMINATED` | Terminated | No | No |
| `DECEASED` | Deceased | No | No |
| `SUSPENDED` | Temporarily suspended | No | No |
| `TRANSFERRED` | Transferred to another unit | No | No |

### Connection to Other Configs

- **HRMS**: New employees are created with status `EMPLOYED`. The status changes when an admin deactivates or transfers them.
- **PGR**: When assigning complaints, only `EMPLOYED` employees with `PGR_LME` role appear in the assignment list.

---

## egov-hrms.DeactivationReason

Reasons for deactivating an employee account.

### Values

| Code | Description |
|------|-------------|
| `OTHERS` | General/unspecified reason |
| `ORDERBYCOMMISSIONER` | Order by commissioner |

### Connection to Other Configs

- **HRMS**: When an admin changes an employee status away from `EMPLOYED`, they must select a deactivation reason from this list.

---

## egov-hrms.Degree

Educational qualifications for the employee profile.

### Example Values

`B.Tech`, `M.Tech`, `BA`, `MA`, `PhD`, `Diploma`, etc.

### Connection to Other Configs

- **HRMS**: Optional field in the employee profile "Education" section.

---

## egov-hrms.Specalization

Professional specialization areas.

### Connection to Other Configs

- **HRMS**: Optional field in the employee profile under education details.

---

## egov-hrms.EmploymentTest

Types of employment tests/exams.

### Connection to Other Configs

- **HRMS**: Optional field in the employee profile.

---

## How Employee Creation Uses These Configs

When creating an employee via HRMS (either API or UI), these MDMS masters are used:

```
Employee Creation requires:
  +-- egov-hrms.EmployeeType ........... "PERMANENT" (required)
  +-- egov-hrms.EmployeeStatus ......... defaults to "EMPLOYED"
  +-- common-masters.Department ........ e.g., "DEPT_36" (required)
  +-- common-masters.Designation ....... e.g., "DESIG_1002" (required)
  +-- common-masters.GenderType ........ e.g., "MALE" (required)
  +-- ACCESSCONTROL-ROLES.roles ........ e.g., ["EMPLOYEE", "GRO"] (required)
  +-- egov-location.TenantBoundary ..... jurisdiction boundary code (required)
```

If any of these master data entries are missing, HRMS rejects the employee creation with a validation error.

---

## Dependency Summary

| Config | Depends On | Required By |
|--------|-----------|-------------|
| All HRMS masters | Nothing | `egov-hrms` (employee creation/management) |
| Employee creation | HRMS masters + Department + Designation + GenderType + Roles + TenantBoundary | PGR (needs employees with GRO/PGR_LME roles) |
