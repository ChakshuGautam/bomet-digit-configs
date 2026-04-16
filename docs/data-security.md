# Data Security

DIGIT encrypts citizen PII (personally identifiable information) at rest and masks it on display. Four MDMS schemas work together to control this: SecurityPolicy defines what to protect, EncryptionPolicy defines what to encrypt, MaskingPatterns defines how to mask, and DecryptionABAC provides fine-grained access control.

All four are read by `egov-enc-service`, which intercepts data going to/from the database.

## Files

| File | Schema Code | Records | Purpose |
|------|------------|---------|---------|
| `mdms/DataSecurity/SecurityPolicy.json` | `DataSecurity.SecurityPolicy` | 32 models | Defines PII fields and role-based visibility |
| `mdms/DataSecurity/EncryptionPolicy.json` | `DataSecurity.EncryptionPolicy` | 3 models | Defines which fields to encrypt at rest |
| `mdms/DataSecurity/MaskingPatterns.json` | `DataSecurity.MaskingPatterns` | 8 patterns | Regex patterns for PII masking |
| `mdms/DataSecurity/DecryptionABAC.json` | `DataSecurity.DecryptionABAC` | 25 entries | Attribute-Based Access Control for decryption |

---

## DataSecurity.SecurityPolicy

The central config. Defines every data model that contains PII, lists its sensitive fields, and specifies which roles can see the decrypted values.

### Record Structure (User model)

```json
{
  "model": "User",
  "attributes": [
    {
      "name": "name",
      "jsonPath": "name",
      "patternId": "002",
      "defaultVisibility": "PLAIN"
    },
    {
      "name": "mobileNumber",
      "jsonPath": "mobileNumber",
      "patternId": "001",
      "defaultVisibility": "PLAIN"
    }
  ],
  "uniqueIdentifier": {
    "name": "uuid",
    "jsonPath": "/uuid"
  },
  "roleBasedDecryptionPolicy": [
    {
      "roles": ["INTERNAL_MICROSERVICE_ROLE"],
      "attributeAccessList": [
        { "attribute": "name", "firstLevelVisibility": "PLAIN", "secondLevelVisibility": "PLAIN" },
        { "attribute": "mobileNumber", "firstLevelVisibility": "PLAIN", "secondLevelVisibility": "PLAIN" }
      ]
    },
    {
      "roles": ["EMPLOYEE", "GRO", "PGR_LME", "DGRO", "CSR", "SUPERUSER", "PGR_VIEWER", "MDMS_ADMIN"],
      "attributeAccessList": [
        { "attribute": "name", "firstLevelVisibility": "PLAIN", "secondLevelVisibility": "PLAIN" }
      ]
    }
  ]
}
```

### Field Reference

| Field | Description |
|-------|-------------|
| `model` | Data model name. Must match what the service sends to `egov-enc-service`. |
| `attributes[].name` | Field name (used in roleBasedDecryptionPolicy references). |
| `attributes[].jsonPath` | JSON path to the field in the data object. |
| `attributes[].patternId` | Which masking pattern to apply (references `MaskingPatterns`). |
| `attributes[].defaultVisibility` | Default visibility when no role-specific policy matches. `PLAIN` = visible, `MASKED` = masked. |
| `uniqueIdentifier` | Field that uniquely identifies each record (for encryption key lookup). |
| `roleBasedDecryptionPolicy` | Array of role-to-visibility mappings. If a user's role appears here, they see data according to the `attributeAccessList`. |

### Models Relevant to PGR

| Model | PII Fields | Purpose |
|-------|-----------|---------|
| `User` | name, mobileNumber, emailId, username, pan, aadhaarNumber, guardian, addresses | Citizen and employee user records |
| `UserSelf` | Same as User | User viewing their own data |
| `EmployeeReport` | name | Employee names in PGR reports |
| `GROPerformanceReport` | employee | GRO names in performance reports |
| `LMEPerformanceReport` | employee | LME names in performance reports |

### Role-Based Decryption for the User Model

This is critical for PGR. When a GRO searches for employees to assign a complaint, `egov-enc-service` checks if the GRO's role has decryption access:

| Roles | Fields Visible | Purpose |
|-------|---------------|---------|
| `INTERNAL_MICROSERVICE_ROLE` | All 12 fields (PLAIN) | Service-to-service calls |
| `REINDEXING_ROLE` | No fields (empty list) | ES reindexing |
| `WS_*/SW_*` roles | No fields (empty list) | Water/Sewerage (not used in CCRS) |
| `EMPLOYEE`, `GRO`, `PGR_LME`, `DGRO`, `CSR`, `SUPERUSER`, `PGR_VIEWER`, `MDMS_ADMIN` | name, mobileNumber, emailId, username, correspondenceAddress, guardian, fatherOrHusbandName | PGR employees need to see citizen/employee details |

**Without PGR roles in this list**: When a GRO views the complaint assignment screen, employee names appear as `B*XXXXXXXXX` because `egov-enc-service` has no decryption policy for `GRO` and applies the masking pattern.

### Legacy Models

32 models are defined, but most are for modules not used in CCRS (Property Tax, Trade License, Water & Sewerage, Building Permits, Fire NOC). These don't cause issues but are unnecessary configuration bloat.

---

## DataSecurity.EncryptionPolicy

Defines which fields in which data models should be encrypted before writing to PostgreSQL.

### Record Structure

```json
{
  "key": "User",
  "attributeList": [
    { "type": "Normal", "jsonPath": "name" },
    { "type": "Normal", "jsonPath": "mobileNumber" },
    { "type": "Normal", "jsonPath": "emailId" },
    { "type": "Normal", "jsonPath": "username" },
    { "type": "Normal", "jsonPath": "pan" },
    { "type": "Normal", "jsonPath": "aadhaarNumber" },
    { "type": "Normal", "jsonPath": "guardian" },
    { "type": "Normal", "jsonPath": "permanentAddress/address" },
    { "type": "Normal", "jsonPath": "correspondenceAddress/address" }
  ]
}
```

### Models with Encryption

| Key | Fields Encrypted | Used by PGR? |
|-----|-----------------|--------------|
| `User` | name, mobileNumber, emailId, username, pan, aadhaarNumber, guardian, addresses | Yes |
| `UserSearchCriteria` | userName, name, mobileNumber, aadhaarNumber, pan | Yes |
| `BndDetail` | mobileno, emailid, aadharno, icdcode | No |

### How It Works

When `egov-user` saves a user record, `egov-enc-service` intercepts the data, finds matching `EncryptionPolicy` entries, and encrypts the listed fields using AES-256. The database stores ciphertext. On read, the reverse happens -- fields are decrypted, then `SecurityPolicy` determines if the requesting user's role can see the plaintext or gets the masked version.

---

## DataSecurity.MaskingPatterns

Regex patterns used to mask PII when a user's role doesn't have full decryption access.

### Patterns

| Pattern ID | Regex | Effect | Used For |
|-----------|-------|--------|----------|
| `001` | `.(?=.{4})` | Masks all characters except last 4 | Phone numbers: `XXXXXX3456` |
| `002` | `\B[a-zA-Z0-9]` | Masks non-boundary alphanumerics | Names: `J***n D**` |
| `003` | `.(?=.{2})` | Masks all except last 2 | Short identifiers |
| `004` | `(?<=.)[^@\n](?=[^@\n]*?@)\|...` | Masks email characters before @ and after domain start | Emails: `j***@g****.com` |
| `005` | `[A-Za-zÀ-ȕ0-9(),-_., ]` | Matches all characters (full mask) | Addresses |
| `006` | `\w(?=(?:[ \w]*\w){2}$)` | Masks middle words | Multi-word names |
| `007` | `(?<=.).(?=.{3})` | Masks middle, preserves first and last 3 | Medium-length identifiers |
| `008` | `(?<=.).(?=.{2})` | Masks middle, preserves first and last 2 | Short identifiers |

### Connection to SecurityPolicy

Each attribute in `SecurityPolicy` references a `patternId`. When a role lacks decryption access, the pattern is applied:
- `name` uses pattern `002` (partial mask): `Kamau N***`
- `mobileNumber` uses pattern `001` (last-4 visible): `XXXXXX7890`
- `emailId` uses pattern `004` (email mask): `k***@g***l.com`
- `address` uses pattern `005` (full mask): `XXXXXXXXXX`

---

## DataSecurity.DecryptionABAC

Attribute-Based Access Control for decryption. Provides fine-grained control beyond role-based policies.

### Record Structure

```json
{
  "key": "ALL_ACCESS",
  "roleAttributeAccessList": [
    {
      "roleCode": "CITIZEN",
      "attributeAccessList": [
        {
          "attribute": { "jsonPath": "*/name" },
          "accessType": "PLAIN"
        },
        {
          "attribute": { "jsonPath": "*/mobileNumber" },
          "accessType": "PLAIN"
        }
      ]
    }
  ]
}
```

### Access Keys

| Key | Roles | Description |
|-----|-------|-------------|
| `ALL_ACCESS` | SYSTEM, CITIZEN, ANONYMOUS + many module-specific roles | Wildcard access for basic fields |

### How ABAC Differs from SecurityPolicy

- **SecurityPolicy**: "Can role X see field Y in model Z?" -- Model-level control.
- **DecryptionABAC**: "Can role X see field pattern `*/name` across any model?" -- Cross-model control using JSON path wildcards.

ABAC is checked when the specific model's SecurityPolicy doesn't have a matching entry. It acts as a fallback.

---

## Dependency Chain

```
MaskingPatterns (standalone, no dependencies)
    ^
    | referenced by patternId
    |
SecurityPolicy (depends on MaskingPatterns + Roles)
    ^
    | enforced by
    |
EncryptionPolicy (standalone, no dependencies)
    |
    v
egov-enc-service reads all four at runtime
    |
    v
DecryptionABAC (fallback for SecurityPolicy, depends on Roles)
```

**Required by**: `egov-enc-service`, which intercepts all read/write operations containing PII.

**Depends on**: `ACCESSCONTROL-ROLES.roles` (role codes referenced in policies).
