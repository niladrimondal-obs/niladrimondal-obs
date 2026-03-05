# SQL Server AlwaysOn HA Monitoring — Dynatrace Custom Extension

> **Production-grade SQL Server Availability Group replica role monitoring with custom topology, ServiceNow CI resolution, and automated alerting.**

---

## The Problem This Solves

Most teams monitor SQL Server AlwaysOn Availability Groups at the query level — they know *something* failed, but have no structured way to detect a **PRIMARY → SECONDARY replica role flip** and route it to the right team via ITSM.

When we first implemented alerting on this metric, we hit a critical gap:

> ❌ The alert fired against a **generic platform entity**, not the actual SQL DB node.  
> ❌ ServiceNow CI resolution failed — the ticket had no CI attached.  
> ❌ The on-call team couldn't identify the affected node from the alert alone.

The fix wasn't just a query tweak. It required **rethinking the topology** — creating a custom entity type, binding the metric to it, and using that entity as both a dimension and the alert source. Once done:

> ✅ Every alert fires against the correct SQL DB Node entity  
> ✅ ServiceNow CI resolution works — ticket auto-attaches to the right CI  
> ✅ The affected node, replica role, and availability group are visible directly on the alert  

This README documents the full architecture and implementation so you can replicate it for any application landscape.

---

## Architecture Overview

```
SQL Server AlwaysOn Cluster
        │
        │  SQL Query (every 5 min)
        ▼
┌─────────────────────────────────┐
│   Dynatrace Extension (EF2.0)   │
│   - Runs query against view     │
│   - Extracts replica role       │
│   - Maps role → numeric metric  │
│     PRIMARY   = 1               │
│     SECONDARY = 2               │
│     UNKNOWN   = 0               │
└────────────┬────────────────────┘
             │ Publishes metric
             ▼
┌─────────────────────────────────┐
│   Custom Metric                 │
│   {app_code}.replicarole        │
│   Dimensions:                   │
│   - availabilitygroup           │
│   - sql.cluster.node.name       │  ◄── Used as entity ID
│   - replica.role                │
│   - listener.name               │
│   - landscape                   │
│   - dt.source_entity ◄──────────┼── Binds metric to Custom Entity
└────────────┬────────────────────┘
             │ Creates / updates
             ▼
┌─────────────────────────────────┐
│   Custom Topology Entity        │
│   Type: custom:sql-db-node      │
│   ID Pattern: sql.cluster.node  │
│   Attributes: AG, landscape     │
└────────────┬────────────────────┘
             │ Referenced in
             ▼
┌─────────────────────────────────┐
│   Davis Alerting Rule           │
│   Threshold: role_id > 1        │
│   (fires when PRIMARY flips     │
│    to SECONDARY or UNKNOWN)     │
│   Source Entity: sql-db-node    │
└────────────┬────────────────────┘
             │ Triggers
             ▼
┌─────────────────────────────────┐
│   ServiceNow ITSM               │
│   CI resolved from entity       │
│   Incident auto-created with    │
│   correct CI attached           │
└─────────────────────────────────┘
```

---

## Three Key Design Decisions — and Why They Matter

### 1. Creating a Custom Dynamic Entity (Topology)

Out of the box, Dynatrace has no entity type for "SQL DB Node" in an AlwaysOn cluster context. If you publish a custom metric without a topology, Dynatrace attaches it to a generic host or platform entity — which breaks any downstream ITSM integration that relies on CI resolution.

By defining a `custom:sql-db-node` entity type in the extension's topology block:

```yaml
topology:
  types:
    - name: "custom:sql-db-node"
      displayName: "SQL DB Node Custom"
      enabled: true
      rules:
        - idPattern: "{sql.cluster.node.name}"
          instanceNamePattern: "{sql.cluster.node.name}"
          iconPattern: "database"
          sources:
            - sourceType: "Metrics"
              condition: "$prefix({app_code}.replicarole)"
```

Dynatrace now **auto-creates and maintains** one entity per SQL cluster node — dynamically, as metrics flow in. No manual entity management. No scripting. The entity lifecycle is driven by the metric itself.

**Why this matters for your ITSM integration:** ServiceNow ITOM expects a known CI to attach incidents to. A generic platform entity doesn't map to any CI. A `custom:sql-db-node` entity *can* be mapped — and once it is, every alert raised against it carries the CI reference automatically.

---

### 2. The Metric Is Bound to the Entity via `dt.source_entity`

This is the critical link that most implementations miss.

Publishing a metric with custom dimensions is straightforward. But without telling Dynatrace *which entity* the metric belongs to, the metric floats — it exists in the metric store but has no entity context. Alerts raised on it carry no CI.

The `dt.source_entity` dimension solves this:

```yaml
dimensions:
  - key: "dt.source_entity"
    value: "var:{SQL DB Node Custom}"
```

This instructs the extension to resolve the entity reference at ingest time and bind the metric data point directly to the `custom:sql-db-node` entity that matches the current node. The result: **every metric data point is entity-aware from the moment it's written**.

When an alert fires, Dynatrace knows exactly which entity caused it — and that entity is what gets passed to ServiceNow for CI lookup.

---

### 3. Dimensions Chosen — and the Reasoning Behind Each

| Dimension | Source | Why It's Here |
|---|---|---|
| `availabilitygroup` | SQL query column | Identifies the AG — multiple AGs can share nodes; this prevents alert ambiguity |
| `sql.cluster.node.name` | SQL query column | Used as the **entity ID pattern** — this IS the entity. Must be unique and stable |
| `replica.role` | SQL query column | Human-readable role label (PRIMARY/SECONDARY) — for alert descriptions and dashboards |
| `listener.name` | SQL query column | The AG listener DNS name — useful for connection-level troubleshooting |
| `landscape` | Constant: `{APPLICATION_NAME}_UAT` | Separates UAT from PROD metrics in the same tenant — critical for multi-environment setups |
| `dt.source_entity` | Variable: entity reference | **The binding dimension** — connects the metric to the custom topology entity |

The `landscape` constant dimension is worth highlighting separately: in a multi-tenant or multi-environment Dynatrace deployment, the same metric key will receive data from UAT, PROD, and DR environments. Without a constant `landscape` dimension, your dashboards and alerts can't filter by environment. Adding it as a constant — not a query column — means zero query overhead and guaranteed consistency.

---

## Full Implementation

### Step 1 — Database View

The extension queries a SQL view. Below is the expected schema. Adapt the view name and column names to your environment.

```sql
-- Expected view schema (adapt to your environment)
-- View name: replace with your monitoring view
SELECT
    [Availability Group],       -- AG name (string)
    [SQL cluster node name],    -- Fully-qualified node\instance (string, unique)
    [Replica Role],             -- 'PRIMARY' or 'SECONDARY' (string)
    [Listener Name]             -- AG listener DNS name (string)
FROM {your_monitoring_view}
ORDER BY [SQL cluster node name]
```

---

### Step 2 — Extension YAML

Save as `extension.yaml` in your extension package. Replace all `{placeholder}` values.

```yaml
name: "custom:db.query.{app_code}-{environment}"
minDynatraceVersion: "1.303.0"
author:
  name: "{Your Team Name}"
version: "0.0.1"

sqlServer:
  - group: "DBReplicaRole"
    query: "SELECT
        [Availability Group],
        [SQL cluster node name],
        [Replica Role],
        [Listener Name],
        CASE
            WHEN [Replica Role] = 'PRIMARY' THEN 1
            WHEN [Replica Role] = 'SECONDARY' THEN 2
            ELSE 0
        END AS role_id
      FROM {your_monitoring_view}
      ORDER BY [SQL cluster node name]"

    metrics:
      - key: "{app_code}.replicarole"
        value: "col:role_id"
        type: "gauge"

    interval:
      minutes: 5
    timeout: "120"
    featureSet: "DBReplicaRole"

    dimensions:
      - key: "availabilitygroup"
        value: "col:Availability Group"
      - key: "sql.cluster.node.name"
        value: "col:SQL cluster node name"
      - key: "replica.role"
        value: "col:Replica Role"
      - key: "listener.name"
        value: "col:Listener Name"
      - key: "landscape"
        value: "const:{APPLICATION_NAME}_{ENVIRONMENT}"   # e.g. MYAPP_UAT or MYAPP_PROD
      - key: "dt.source_entity"
        value: "var:{SQL DB Node Custom}"

topology:
  types:
    - name: "custom:sql-db-node"
      displayName: "SQL DB Node Custom"
      enabled: true
      rules:
        - idPattern: "{sql.cluster.node.name}"
          instanceNamePattern: "{sql.cluster.node.name}"
          iconPattern: "database"
          sources:
            - sourceType: "Metrics"
              condition: "$prefix({app_code}.replicarole)"
          attributes:
            - key: "availabilitygroup"
              pattern: "{availabilitygroup}"
            - key: "landscape"
              pattern: "{landscape}"
            - key: "Application"
              pattern: "{APPLICATION_NAME}_{ENVIRONMENT}"
```

---

### Step 3 — Davis Alerting Rule (DQL)

Navigate to: **Davis AI → Custom Events for Alerting → Create metric event**

Use this query in the metric selector:

```
timeseries status = avg({app_code}.replicarole),
by: {
  availabilitygroup,
  sql.cluster.node.name,
  listener.name,
  `dt.entity.custom:sql-db-node`
},
interval: 1m
| filter sql.cluster.node.name == "{SQL_NODE_HOSTNAME}\\{INSTANCE_NAME}"
```

**Alert configuration:**

| Parameter | Value | Reasoning |
|---|---|---|
| Threshold | `> 1` | PRIMARY = 1 (healthy). Any value above 1 means role has flipped |
| Violating samples | `2` | Prevents false alerts on transient query delays |
| Sliding window | `7` | 7-minute window gives enough data points at 1m interval |
| Dealerting window | `7` | Matches alerting window — symmetric hysteresis |
| Frequent issue detection | OFF | Role flips are always significant — suppress nothing |

---

### Step 4 — Alert Event Properties

In the alert rule's **Event Properties** section, add:

| Property Key | Value |
|---|---|
| `dt.source_entity` | `{dims:dt.entity.custom:sql-db-node}` |
| `event.type` | `AVAILABILITY` |
| `event.name` | `SQL AG Replica Role Change — {APPLICATION_NAME}` |
| `event.description` | `Node {dims:sql.cluster.node.name} in AG {dims:availabilitygroup} has flipped from PRIMARY. Current role_id: {value}. Landscape: {dims:landscape}` |

The `dt.source_entity` property is what tells ServiceNow ITSM which CI to attach the incident to. Without this, CI resolution fails silently.

---

## What the DB Team Gets

| Capability | Before This Extension | After This Extension |
|---|---|---|
| Replica role visibility | Manual SQL query or SSMS check | Real-time metric in Dynatrace dashboard |
| PRIMARY flip detection | Reactive — discovered after user impact | Proactive — alert fires within 2 minutes of flip |
| Alert → CI linkage in ServiceNow | Broken — generic platform entity, no CI | Working — incident auto-attaches to correct SQL DB Node CI |
| Multi-environment separation | Mixed data, hard to filter | `landscape` dimension separates UAT / PROD / DR cleanly |
| On-call context on alert | Node name only | Node + AG + listener + role + landscape all on the alert |
| False positive rate | High (single sample threshold) | Low (2 violating samples in 7-min window) |

---

## Parameters Reference

| Placeholder | Replace With | Example |
|---|---|---|
| `{app_code}` | Short application identifier, lowercase | `myapp` |
| `{environment}` | Environment label | `uat`, `prod`, `dr` |
| `{APPLICATION_NAME}` | Full application name, uppercase | `MYAPP` |
| `{ENVIRONMENT}` | Environment label, uppercase | `UAT`, `PROD` |
| `{your_monitoring_view}` | Your SQL monitoring view name | `vw_mon_availability` |
| `{SQL_NODE_HOSTNAME}` | Fully-qualified SQL Server hostname | `SQLNODE01` |
| `{INSTANCE_NAME}` | SQL Server named instance | `SQLINSTANCE01` |
| `{Your Team Name}` | Team or author name for extension metadata | `Platform Engineering` |

---

## Prerequisites

- Dynatrace tenant version **1.303.0+**
- Dynatrace Extension Framework 2.0 (EF2.0)
- ActiveGate with SQL Server monitoring enabled
- SQL Server monitoring user with `SELECT` permission on the monitoring view
- ServiceNow ITSM integration configured in Dynatrace (for CI resolution)

---

## Lessons Learned

**Don't skip the topology block.** It feels optional until you need ServiceNow CI resolution — then it's critical. Add it from day one, even if your ITSM integration isn't live yet.

**`dt.source_entity` must be in both the extension dimensions AND the alert event properties.** The extension dimension creates the binding. The alert property passes it downstream to ITSM. One without the other breaks the chain.

**Use a constant `landscape` dimension for every multi-environment extension.** The 2 minutes it takes to add saves hours of dashboard confusion when UAT and PROD metrics start mixing in the same tenant.

---

## Related Portfolio Projects

- [Daily Incident Digest Tool](../incident-digest/) — Automated DT problem summary via Python
- [Auto-Remediation Workflow](../auto-remediation/) — AutomationEngine + ServiceNow full-loop remediation
- [DQL Cookbook](../dql-cookbook/) — Production-grade Grail query library

---



