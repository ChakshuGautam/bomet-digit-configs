# Boundary Configuration

Boundaries define the geographic hierarchy of a government entity. For Bomet County, this is County > SubCounty > Ward. Boundaries are used for HRMS jurisdiction assignment and PGR complaint location tagging.

## Files

| File | Schema Code | Purpose |
|------|------------|---------|
| `mdms/egov-location/TenantBoundary.json` | `egov-location.TenantBoundary` | HRMS jurisdiction hierarchy tree |
| `mdms/CRS-ADMIN-CONSOLE/adminSchema.json` | `CRS-ADMIN-CONSOLE.adminSchema` | Admin console boundary data schema |

---

## egov-location.TenantBoundary

A hierarchical tree of administrative boundaries. Used by HRMS to validate employee jurisdiction assignments.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "boundary": {
      "id": 62797,
      "code": "ke.bomet",
      "name": "Bomet County",
      "label": "County",
      "children": [
        {
          "code": "BOMET_BOMET_CENTRAL",
          "name": "Central",
          "label": "SubCounty",
          "children": [
            {
              "code": "BOMET_BOMET_CENTRAL_CHESOEN",
              "name": "Central Chesoen",
              "label": "Ward",
              "children": []
            }
          ]
        }
      ]
    },
    "hierarchyType": {
      "code": "ADMIN",
      "name": "ADMIN"
    },
    "tenantId": "ke.bomet"
  }
}
```

### Hierarchy Levels

| Level | Label | Example | Description |
|-------|-------|---------|-------------|
| 1 | County | Bomet County | Root administrative unit. Code must match tenant code (`ke.bomet`). |
| 2 | SubCounty | Central, Chepalungu, Konoin, Sotik, Bomet East | Second-level divisions |
| 3 | Ward | Central Chesoen, Central Mutarakwa, etc. | Lowest-level administrative unit |

### Bomet Boundary Tree

```
Bomet County (ke.bomet)
+-- Bomet Central (BOMET_BOMET_CENTRAL)
|   +-- Central Chesoen (BOMET_BOMET_CENTRAL_CHESOEN)
|   +-- Central Mutarakwa (BOMET_BOMET_CENTRAL_MUTARAKWA)
|   +-- Central Nadaraweta (BOMET_BOMET_CENTRAL_NADARAWETA)
|   +-- Central Silibwet Township (BOMET_BOMET_CENTRAL_SILIBWET_TOWNSHIP)
|   +-- Central Singorwet (BOMET_BOMET_CENTRAL_SINGORWET)
|
+-- Bomet East (BOMET_BOMET_EAST)
|   +-- (5 wards)
|
+-- Chepalungu (BOMET_CHEPALUNGU)
|   +-- (5 wards)
|
+-- Konoin (BOMET_KONOIN)
|   +-- (5 wards)
|
+-- Sotik (BOMET_SOTIK)
    +-- (5 wards)
```

**Total**: 5 subcounties, 25 wards.

### Critical: Root Boundary Code

The root boundary's `code` field **must** match the tenant code (e.g., `ke.bomet`). This is not optional.

**Why**: The HRMS employee creation UI sends the tenant code from the `tenant.tenants` dropdown as the jurisdiction boundary value. The HRMS backend validates:

```java
// EmployeeValidator.java:479
$.TenantBoundary[?(@.boundary.code == "ke.bomet")].hierarchyType.code
```

If the root boundary code is something else (e.g., `"COUNTY"`), this JSON path query finds nothing, and HRMS rejects the employee with `ERR_HRMS_INVALID_JURISDICTION_HEIRARCHY`.

### Code Convention

Boundary codes follow the pattern: `<COUNTY>_<SUBCOUNTY>_<WARD>` in SCREAMING_SNAKE_CASE:
- County: `ke.bomet` (special: matches tenant code)
- SubCounty: `BOMET_BOMET_CENTRAL`
- Ward: `BOMET_BOMET_CENTRAL_CHESOEN`

### Localization

Boundary codes and type labels need localization messages for the UI to display human-readable names:

| Key Pattern | Example | Purpose |
|------------|---------|---------|
| `EGOV_LOCATION_BOUNDARYTYPE_<LEVEL>` | `EGOV_LOCATION_BOUNDARYTYPE_WARD` → "Ward" | Boundary type label |
| `ADMIN_<LEVEL>` | `ADMIN_SUBCOUNTY` → "Sub County" | Alternative type label |
| `<BOUNDARY_CODE>` | `BOMET_BOMET_CENTRAL` → "Central" | Boundary name display |

Without these localizations, the UI shows raw codes like `ADMIN_SUBCOUNTY` and `BOMET_BOMET_CENTRAL_CHESOEN`.

### Connection to Other Configs

- **HRMS**: Employee jurisdiction assignment references boundary codes from this tree.
- **PGR**: Complaint `address.locality.code` references ward-level codes.
- **INBOX**: The "area" filter matches against locality codes.
- **Localization**: Boundary codes need localization messages for display.
- **tenant.tenants**: Root boundary code must match tenant code.

---

## CRS-ADMIN-CONSOLE.adminSchema

Schema definition for the admin console's boundary data management UI.

### Record Structure

```json
{
  "tenantId": "statea",
  "data": {
    "title": "CRS_BOUNDARY_DATA",
    "properties": {
      "numberProperties": [
        { "name": "CRS_LAT", "type": "number", "isRequired": true, "description": "Latitude", "orderNumber": 2 },
        { "name": "CRS_LONG", "type": "number", "isRequired": true, "description": "Longitude", "orderNumber": 3 }
      ],
      "stringProperties": [
        { "name": "CRS_BOUNDARY_CODE", "type": "string", "isRequired": true, "description": "Boundary Code", "orderNumber": 1, "freezeColumn": true }
      ]
    },
    "campaignType": "all"
  }
}
```

This defines the spreadsheet columns in the admin console's boundary data editor:
1. Boundary Code (string, frozen column)
2. Latitude (number)
3. Longitude (number)

### Connection to Other Configs

- **DIGIT UI Admin Console**: Renders the boundary data editor form from this schema.

---

## Dependency Summary

| Config | Depends On | Required By |
|--------|-----------|-------------|
| `TenantBoundary` | `tenant.tenants` (root code match) | HRMS (jurisdiction), PGR (locality), INBOX (area filter) |
| `adminSchema` | Nothing | Admin console UI |
