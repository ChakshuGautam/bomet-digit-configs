# PGR Configuration (Public Grievance Redressal)

PGR-specific configs define what types of complaints citizens can file and UI behavior settings.

## Files

| File | Schema Code | Purpose |
|------|------------|---------|
| `mdms/RAINMAKER-PGR/ServiceDefs.json` | `RAINMAKER-PGR.ServiceDefs` | Complaint type definitions |
| `mdms/RAINMAKER-PGR/UIConstants.json` | `RAINMAKER-PGR.UIConstants` | PGR UI behavior settings |

---

## RAINMAKER-PGR.ServiceDefs

The complaint type catalog. Each record defines one type of complaint citizens can file (e.g., "Blocked sewage", "No streetlight"). This is the most frequently customized config for new deployments.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "serviceCode": "BurningOfGarbage",
    "name": "Burning of garbage",
    "active": true,
    "keywords": "garbage, burn, fire, health, waste, smoke, plastic, illegal",
    "menuPath": "Garbage",
    "slaHours": 336,
    "department": "DEPT_3",
    "order": 1
  }
}
```

### Field Reference

| Field | Required | Description |
|-------|----------|-------------|
| `serviceCode` | Yes | Unique identifier (PascalCase, no spaces). Used in API calls and localization keys. |
| `name` | Yes | Human-readable name shown in the complaint filing form. |
| `active` | Yes | Whether this complaint type is available. Set `false` to hide without deleting. |
| `keywords` | Yes | Comma-separated search keywords. Used by the citizen UI for complaint type search/autocomplete. **Must be a plain string, not a JSON array.** |
| `menuPath` | No | Category grouping in the complaint type selector (e.g., "Garbage", "WaterandSewage", "RoadsAndFootpaths"). Only used at the root tenant level. |
| `slaHours` | Yes | SLA deadline in hours. 336 hours = 14 days, 504 hours = 21 days. The inbox shows red/amber/green based on this. |
| `department` | Yes | Department code from `common-masters.Department` that handles this complaint type. |
| `order` | No | Display order within the category. Lower numbers appear first. |

### Dual-Level Registration

ServiceDefs must exist at **both** the root tenant AND the city tenant:

| Level | Purpose | Example `tenantId` |
|-------|---------|-------------------|
| Root (`ke`) | PGR-services resolves complaint types here when processing API calls | `ke` |
| City (`ke.bomet`) | The UI filters complaint types for the city-level dropdown | `ke.bomet` |

If only registered at root: the UI complaint type dropdown may be empty.
If only registered at city: `pgr-services` returns "Failed to parse mdms response for service".

### Bomet Complaint Types (ke tenant — 47 types)

#### Garbage (DEPT_3)
| Service Code | Name | SLA |
|-------------|------|-----|
| `BurningOfGarbage` | Burning of garbage | 336h |
| `DamagedGarbageBin` | Damaged garbage bin | 336h |
| `GarbageNeedsTobeCleared` | Garbage needs to be cleared | 336h |

#### Water & Sewage (DEPT_4)
| Service Code | Name | SLA |
|-------------|------|-----|
| `BlockOrOverflowingSewage` | Block / Overflowing sewage | 336h |
| `BrokenWaterPipeOrLeakage` | Broken water pipe / Leakage | 336h |
| `DirtyWaterSupply` | Dirty water supply | 336h |
| `illegalDischargeOfSewage` | Illegal discharge of sewage | 336h |
| `NoWaterSupply` | No water supply | 336h |
| `WaterPressureisVeryLow` | Water pressure is very low | 336h |

#### Roads & Footpaths (DEPT_4)
| Service Code | Name | SLA |
|-------------|------|-----|
| `ConstructionMaterialLyingOntheRoad` | Construction material lying on the road | 336h |
| `DamagedOrBlockedFootpath` | Damaged/blocked footpath | 336h |
| `DamagedRoad` | Damaged road | 336h |
| `RequestforNewStreetlight` | Request for new streetlight | 336h |

#### Trees (DEPT_5)
| Service Code | Name | SLA |
|-------------|------|-----|
| `CuttingOrTrimmingOfTreeRequired` | Cutting/trimming of tree required | 336h |
| `IllegalCuttingOfTrees` | Illegal cutting of trees | 336h |

#### Land Violations (DEPT_6)
| Service Code | Name | SLA |
|-------------|------|-----|
| `IllegalConstructions` | Illegal constructions | 336h |
| `IllegalParking` | Illegal parking | 336h |
| `IllegalShopsOnFootpath` | Illegal shops on footpath | 336h |

#### Health Services (DEPT_36 — Bomet-specific)
Bomet County has additional health-specific complaint types not found in the default DIGIT deployment. These are mapped to the custom `DEPT_36` (HealthServices) department.

#### Other Categories
Animals (dead animals, stray dogs/cattle), Public Toilets, Streetlights, Open Defecation, Mosquito/Pest Control, and general categories.

### Localization Keys

For each `serviceCode`, the UI expects localization messages:

| Key Pattern | Example | Purpose |
|------------|---------|---------|
| `SERVICEDEFS.{SERVICE_CODE}` | `SERVICEDEFS.BurningOfGarbage` | Complaint type label in filing form |
| `SERVICEDEFS.RAINMAKER-PGR.{SERVICE_CODE}` | `SERVICEDEFS.RAINMAKER-PGR.BurningOfGarbage` | Alternative key pattern |
| `CATEGORY_{MENU_PATH}` | `CATEGORY_Garbage` | Category group label |

### Connection to Other Configs

- **common-masters.Department**: `department` field references a department code.
- **Localization**: Display names come from localization messages, not the `name` field directly.
- **Workflow**: SLA from the workflow definition (5 days overall) is separate from per-complaint-type SLA.
- **INBOX**: Inbox filters can group by complaint type/service code.

---

## RAINMAKER-PGR.UIConstants

PGR UI behavior settings.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "REOPENSLA": 432000000
  }
}
```

### Constants

| Key | Value | Unit | Description |
|-----|-------|------|-------------|
| `REOPENSLA` | 432000000 | milliseconds | Time window after resolution during which a citizen can reopen a complaint. 432000000 ms = 5 days. After this window, the "Reopen" button disappears. |

### Connection to Other Configs

- **DIGIT UI**: The PGR complaint detail page reads this to determine whether to show the "Reopen" button.
- **Workflow**: The workflow allows REOPEN action at any time, but the UI hides the button after `REOPENSLA` expires.

---

## Dependency Summary

| Config | Depends On | Required By |
|--------|-----------|-------------|
| `ServiceDefs` | `common-masters.Department`, MDMS schemas | `pgr-services` (complaint creation), DIGIT UI (complaint type selector) |
| `UIConstants` | Nothing | DIGIT UI (PGR behavior) |
