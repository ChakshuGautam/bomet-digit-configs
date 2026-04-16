# Inbox Configuration

The employee inbox (where GROs and LMEs see their work queue) is powered by Elasticsearch via `inbox-v2`. This config defines how the inbox queries ES.

## Files

| File | Schema Code | Purpose |
|------|------------|---------|
| `mdms/INBOX/InboxQueryConfiguration.json` | `INBOX.InboxQueryConfiguration` | Elasticsearch query structure for the inbox |

---

## INBOX.InboxQueryConfiguration

Defines the ES index to query, which fields to return, available filters, and sort order.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "index": "inbox-pgr-services",
    "module": "pgr-services",
    "sortBy": {
      "path": "Data.service.auditDetails.createdTime",
      "defaultOrder": "ASC"
    },
    "sourceFilterPathList": [
      "Data.currentProcessInstance",
      "Data.service.serviceRequestId",
      "Data.service.address.locality.code",
      "Data.service.applicationStatus",
      "Data.service.citizen",
      "Data.service.auditDetails.createdTime",
      "Data.auditDetails",
      "Data.tenantId"
    ],
    "allowedSearchCriteria": [...]
  }
}
```

### Key Fields

| Field | Description |
|-------|-------------|
| `index` | Elasticsearch index name. `inbox-pgr-services` is populated by `egov-indexer` from PGR complaint events. |
| `module` | Service module name. Must match the business service in the workflow. |
| `sortBy.path` | Default sort field. Complaints sorted by creation time. |
| `sortBy.defaultOrder` | Sort direction (`ASC` = oldest first). |
| `sourceFilterPathList` | ES `_source` filter -- which fields to return from each document. Reduces payload size. |
| `allowedSearchCriteria` | Available filter/search parameters in the inbox UI. |

### Search Criteria (Filters)

These are the filters available in the employee inbox:

| Name | ES Path | Operator | Description |
|------|---------|----------|-------------|
| `area` | `Data.service.address.locality.code.keyword` | EQUAL | Filter by locality/ward |
| `status` | `Data.currentProcessInstance.state.uuid.keyword` | EQUAL | Filter by workflow state |
| `assignedToMe` | `Data.workflow.assignes.*.uuid.keyword` | EQUAL | Show only complaints assigned to current user |
| `fromDate` | `Data.service.auditDetails.createdTime` | GTE | Date range start |
| `toDate` | `Data.service.auditDetails.createdTime` | LTE | Date range end |
| `complaintNumber` | `Data.service.serviceRequestId.keyword` | EQUAL | Search by complaint ID |
| `mobileNumber` | `Data.service.citizen.mobileNumber.keyword` | EQUAL | Search by citizen phone |
| `tenantId` | `Data.service.tenantId.keyword` | EQUAL | Filter by tenant |
| `assignee` | `Data.currentProcessInstance.assignes.uuid.keyword` | EQUAL | Filter by assigned employee UUID |

### How the Inbox Works

```
1. Employee opens inbox in DIGIT UI
2. UI calls inbox-v2 API with filters (area, status, assignedToMe, etc.)
3. inbox-v2 reads InboxQueryConfiguration from MDMS
4. inbox-v2 constructs ES query from config + filters
5. inbox-v2 queries Elasticsearch index "inbox-pgr-services"
6. Results returned to UI as list of complaints with:
   - Service request ID
   - Locality
   - Status
   - Citizen details
   - Creation time
   - Current workflow process instance
```

### Connection to Other Configs

- **Workflow**: The `status` filter maps to workflow state UUIDs. The `currentProcessInstance` contains the current workflow state.
- **Boundary**: The `area` filter uses locality codes from the boundary hierarchy.
- **DataSecurity**: Citizen details in search results are subject to SecurityPolicy masking.
- **Elasticsearch**: The `inbox-pgr-services` index must exist and be populated by `egov-indexer`.

### Without This Config

If `INBOX.InboxQueryConfiguration` is missing or misconfigured:
- The employee inbox shows "No configuration found for module"
- Employees cannot see their work queue
- GROs cannot assign complaints
- PGR_LMEs cannot see complaints assigned to them

---

## Dependency Summary

| Config | Depends On | Required By |
|--------|-----------|-------------|
| `InboxQueryConfiguration` | Elasticsearch index must exist | `inbox-v2` service, DIGIT UI employee inbox |
