# Tenant Configuration

DIGIT is a multi-tenant platform. The tenant config defines which government entities exist and which modules are active for each.

## Files

| File | Schema Code | Purpose |
|------|------------|---------|
| `mdms/tenant/tenants.json` | `tenant.tenants` | Registers each government entity (state root + city) |
| `mdms/tenant/citymodule.json` | `tenant.citymodule` | Activates modules (PGR, HRMS, Workbench) per tenant |

---

## tenant.tenants

Every government entity must be registered here before any other config works. DIGIT uses a two-level hierarchy:

- **Root tenant** (state/country level): `ke`, `pg`, `statea`
- **City tenant** (municipality level): `ke.bomet`, `pg.citya`, `statea.citya`

### Record Structure

```json
{
  "tenantId": "ke",          // root tenant where this record is stored
  "data": {
    "code": "ke.bomet",       // unique tenant code (dot-separated hierarchy)
    "name": "Bomet County",   // human-readable name
    "type": "CITY",           // "CITY" for city tenants, absent for root
    "tenantId": "ke.bomet",   // same as code (legacy field)
    "description": "...",     // optional description
    "city": {
      "code": "BOMET",
      "name": "Bomet County",
      "districtCode": "BOMET",
      "districtName": "Bomet",
      "latitude": -0.7817,
      "longitude": 35.3428
    }
  }
}
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `code` | Yes | Dot-separated tenant code. Root tenants have no dot (e.g., `ke`). City tenants have format `root.city` (e.g., `ke.bomet`). |
| `name` | Yes | Display name shown in UI dropdowns and headers. |
| `type` | No | Set to `"CITY"` for city-level tenants. Omitted for root tenants. |
| `city.code` | Yes | Uppercase code used in ID generation patterns (e.g., `BOMET` in `KE-PGR-2026-04-16-000001`). |
| `city.latitude/longitude` | No | Geographic coordinates for map display. |

### Bomet Tenants

| Code | Name | Type |
|------|------|------|
| `ke` | Kenya | Root |
| `ke.bomet` | Bomet County | City |

### How Other Configs Use This

- **HRMS**: Employee `tenantId` must match a registered city tenant.
- **PGR**: Complaint `tenantId` must match a registered city tenant. `pgr-services` looks up the root tenant to resolve ServiceDefs.
- **UI**: The tenant selector dropdown is populated from this data.
- **ID Generation**: `common-masters.IdFormat` uses `[CITY.CODE]` from the `city.code` field.
- **Workflow**: Workflow definitions are scoped to root tenants but apply to all cities under them.

### Dependency

**Required by**: Every other config. Without tenant registration, MDMS creates at the tenant level fail with "tenant not found".

**Depends on**: Nothing. This is seeded first.

---

## tenant.citymodule

Controls which modules appear in the UI sidebar for each tenant.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "code": "PGR",
    "module": "PGR",
    "order": 2,
    "active": true,
    "tenants": [
      { "code": "ke" },
      { "code": "ke.bomet" }
    ]
  }
}
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `code` | Yes | Module identifier. Must match the UI module name. |
| `module` | Yes | Same as `code` (legacy duplication). |
| `order` | Yes | Display order in the sidebar. Lower numbers appear first. |
| `active` | Yes | Whether the module is enabled. Set to `false` to hide without deleting. |
| `tenants` | Yes | Array of tenant codes this module is active for. Must include both root and city. |

### Modules in Bomet

| Module | Order | Description |
|--------|-------|-------------|
| `PGR` | 2 | Public Grievance Redressal - complaint filing and management |
| `HRMS` | 2 | Human Resource Management - employee creation and management |
| `Workbench` | 13 | MDMS data management UI (admin only, root tenant only) |

### How Other Configs Use This

- **DIGIT UI**: Reads this on login to determine which sidebar items to show. If PGR is not listed, the complaint module is hidden.
- **Admin Console**: Uses this to determine which modules an admin can configure.

### Dependency

**Required by**: DIGIT UI (determines visible modules).

**Depends on**: `tenant.tenants` (tenant codes in the `tenants` array must be registered).
