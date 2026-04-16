# Workflow Configuration

The PGR complaint lifecycle is a state machine managed by `egov-workflow-v2`. MDMS configs supplement the core workflow definition with auto-escalation rules and service flags.

## Files

| File | Schema Code | Purpose |
|------|------------|---------|
| `workflows/business-services.json` | (API export) | Registered PGR workflow instances |
| `mdms/Workflow/BusinessService.json` | `Workflow.BusinessService` | Business service metadata (state machine definitions) |
| `mdms/Workflow/BusinessServiceConfig.json` | `Workflow.BusinessServiceConfig` | Service-level flags (e.g., state-level vs city-level) |
| `mdms/Workflow/AutoEscalation.json` | `Workflow.AutoEscalation` | Automatic escalation timers and actions |
| `mdms/Workflow/AutoEscalationStatesToIgnore.json` | `Workflow.AutoEscalationStatesToIgnore` | States exempt from auto-escalation |
| `mdms/Workflow/BusinessServiceMasterConfig.json` | `Workflow.BusinessServiceMasterConfig` | Master workflow configuration |

---

## PGR Workflow State Machine

The core workflow is registered via the `egov-workflow-v2` API (not MDMS). The `workflows/business-services.json` file shows which tenants have PGR registered:

| Tenant | Business Service | UUID |
|--------|-----------------|------|
| `ciboot` | PGR | `8b7cfb76-...` |
| `ke` | PGR | `6463d60d-...` |
| `pg.citest` | PGR | `02a68157-...` |
| `statea` | PGR | `1c76df7a-...` |

### PGR State Machine

```
                           +--> REJECTED -----> CLOSEDAFTERREJECTION
                           |      ^
                           |      | REOPEN
CITIZEN files              |      v
complaint      +---------->+  PENDINGFORASSIGNMENT
               |           |      |
               |           |      | GRO: ASSIGN
               |           |      v
               |           |  PENDINGATLME ---------> PENDINGFORREASSIGNMENT
               |           |      |                       |
               |           |      | PGR_LME: RESOLVE      | GRO: ASSIGN
               |           |      v                       |
               |           |  RESOLVED ------+            |
               |           |      |          |            v
               |           |      | CITIZEN: |    (back to PENDINGATLME)
               |           |      | RATE     |
               |           |      v          |
               |           |  CLOSEDAFTER    | CITIZEN: REOPEN
               |           |  RESOLUTION <---+
               |           |
               |           | AUTO_ESCALATE: FORWARD
               |           v
               |  PENDINGATSUPERVISOR
               |           |
               |           | SUPERVISOR: RESOLVEBYSUPERVISOR
               |           v
               |  RESOLVEDBYSUPERVISOR
               |
               +--- CANCELLED (terminal)
```

### States

| State | Terminal? | Who Can Act | Description |
|-------|----------|-------------|-------------|
| (initial) | - | `CITIZEN`, `CSR` | Starting state. APPLY action files the complaint. |
| `PENDINGFORASSIGNMENT` | No | `GRO`, `PGR_VIEWER` | Complaint waiting for assignment to a field worker. |
| `PENDINGATLME` | No | `PGR_LME`, `PGR_VIEWER` | Assigned to a field worker, awaiting resolution. |
| `PENDINGFORREASSIGNMENT` | No | `GRO`, `PGR_VIEWER` | Field worker requested reassignment. |
| `PENDINGATSUPERVISOR` | No | `SUPERVISOR` | Auto-escalated to supervisor. |
| `RESOLVED` | Yes* | `CITIZEN`, `CSR`, `PGR_VIEWER` | Resolved by field worker. Citizen can reopen or rate. |
| `REJECTED` | Yes* | `CITIZEN`, `CSR`, `PGR_VIEWER` | Rejected by GRO. Citizen can reopen or rate. |
| `CLOSEDAFTERRESOLUTION` | Yes | - | Citizen rated after resolution. Final state. |
| `CLOSEDAFTERREJECTION` | Yes | - | Citizen rated after rejection. Final state. |
| `RESOLVEDBYSUPERVISOR` | Yes | - | Supervisor resolved an escalated complaint. |
| `CANCELLED` | Yes | - | Complaint cancelled. |

*RESOLVED and REJECTED are terminal but allow REOPEN/RATE actions.

### Actions

| Action | From State | To State | Roles |
|--------|-----------|----------|-------|
| `APPLY` | (initial) | PENDINGFORASSIGNMENT | CITIZEN, CSR |
| `ASSIGN` | PENDINGFORASSIGNMENT | PENDINGATLME | GRO, PGR_VIEWER |
| `ASSIGN` | PENDINGFORREASSIGNMENT | PENDINGATLME | GRO, PGR_VIEWER |
| `REJECT` | PENDINGFORASSIGNMENT | REJECTED | GRO, PGR_VIEWER |
| `REJECT` | PENDINGFORREASSIGNMENT | REJECTED | GRO, PGR_VIEWER |
| `RESOLVE` | PENDINGATLME | RESOLVED | PGR_LME, PGR_VIEWER |
| `REASSIGN` | PENDINGATLME | PENDINGFORREASSIGNMENT | PGR_LME, PGR_VIEWER |
| `REOPEN` | RESOLVED | PENDINGFORASSIGNMENT | CITIZEN, CSR, PGR_VIEWER |
| `REOPEN` | REJECTED | PENDINGFORASSIGNMENT | CITIZEN, CSR, PGR_VIEWER |
| `RATE` | RESOLVED | CLOSEDAFTERRESOLUTION | CITIZEN |
| `RATE` | REJECTED | CLOSEDAFTERREJECTION | CITIZEN |
| `COMMENT` | (multiple) | (same state) | CITIZEN |
| `FORWARD` | PENDINGATLME | PENDINGATSUPERVISOR | AUTO_ESCALATE |
| `ASSIGNEDBYAUTOESCALATION` | PENDINGFORASSIGNMENT | PENDINGATLME | AUTO_ESCALATE |
| `RESOLVEBYSUPERVISOR` | PENDINGATSUPERVISOR | RESOLVEDBYSUPERVISOR | SUPERVISOR |

### Overall SLA

| Field | Value |
|-------|-------|
| Business SLA | 432,000,000 ms (5 days) |
| Per-complaint-type SLA | Defined in `RAINMAKER-PGR.ServiceDefs.slaHours` (typically 336h = 14 days) |

---

## Workflow.AutoEscalation

Defines automatic escalation rules. When a complaint sits in a state beyond its SLA, the system automatically transitions it.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "businessService": "Incident",
    "module": "im-services",
    "state": "REJECTED",
    "action": "CLOSE",
    "topic": "im-auto-escalation",
    "active": "true",
    "stateSLA": 3.0,
    "businessSLA": 3456
  }
}
```

### Field Reference

| Field | Description |
|-------|-------------|
| `businessService` | Which workflow this escalation applies to (e.g., `PGR`, `Incident`) |
| `state` | The state to watch for SLA breach |
| `action` | Action to auto-execute when SLA is breached |
| `topic` | Kafka topic to publish the escalation event |
| `stateSLA` | State-level SLA threshold (days) |
| `businessSLA` | Overall business SLA threshold (ms) |

### Connection to Other Configs

- **Workflow state machine**: Auto-escalation actions must be valid transitions in the state machine.
- **Access Control**: The `AUTO_ESCALATE` role must have permissions for the escalation actions.

---

## Workflow.AutoEscalationStatesToIgnore

States that should not trigger auto-escalation even if SLA expires.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "businessService": "NewTL",
    "module": "TL",
    "state": ["INITIATED", "PENDINGAPPROVAL"]
  }
}
```

Currently only configured for Trade License (`NewTL`), not PGR.

---

## Workflow.BusinessServiceConfig

Flags for workflow business services.

### Record Structure

```json
{
  "tenantId": "ke",
  "data": {
    "code": "FIRENOC",
    "isStateLevel": true
  }
}
```

| Code | isStateLevel | Description |
|------|-------------|-------------|
| `FIRENOC` | true | Fire NOC workflow operates at state level |
| `NEWTL` | true | Trade License workflow operates at state level |

PGR is not listed here because it inherits from the root tenant by default.

---

## Workflow.BusinessService (MDMS)

MDMS-stored business service metadata. Contains the full state machine definition including states, transitions, roles, and SLA per state.

This is a large, detailed config. The `ke` tenant has definitions for:
- NewTL (Trade License)
- PGR (Public Grievance Redressal)
- Various Incident types (Incident, Incident_High, Incident_Low, Incident_Medium)

### Connection to Other Configs

- **ACCESSCONTROL-ROLES.roles**: Roles referenced in transitions must exist.
- **RAINMAKER-PGR.ServiceDefs**: Per-complaint SLA supplements the workflow SLA.
- **common-masters.wfSlaConfig**: SLA traffic light colors in the inbox.

---

## Dependency Summary

| Config | Depends On | Required By |
|--------|-----------|-------------|
| Workflow state machine (API) | `ACCESSCONTROL-ROLES.roles` | `pgr-services` (complaint lifecycle) |
| `AutoEscalation` | Workflow state machine | Auto-escalation cron job |
| `BusinessServiceConfig` | Nothing | Workflow engine (state vs city level routing) |
| `BusinessService` (MDMS) | Roles | DIGIT UI (displays available actions) |
