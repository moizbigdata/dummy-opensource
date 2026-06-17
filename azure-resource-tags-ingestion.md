# Resource tags in Log Analytics (Grafana variables)

Tags on Arc machines (`Company`, `Site`, `ClusterId`, etc.) are **Azure Resource Manager metadata**. They are **not** collected by the Azure Monitor Agent or your Default/Custom DCR.

| Data | Source | Collected by DCR? |
|------|--------|-------------------|
| Perf, Event, Heartbeat | AMA on each node | Yes |
| Resource tags | Azure Resource Graph (ARM) | **No** |

There is **no built-in `AzureResource` table** in Log Analytics. Use one of the options below.

---

## Option A — Recommended: `arg("").Resources` (no DCR)

Query Azure Resource Graph **directly from Log Analytics** using cross-service KQL. No extra ingestion, no custom table.

**Prerequisites**

- Grafana SP: **Log Analytics Reader** on central LAW
- Grafana SP: **Reader** on all site subscriptions (ARG read access)
- Run queries from the central LAW in the Azure portal first to validate

**Example — list companies**

```kusto
arg("").Resources
| where type =~ "microsoft.hybridcompute/machines"
| extend Company = tostring(tags.Company)
| where isnotempty(Company)
| distinct Company
| order by Company asc
```

**Example — cluster RG from tags**

```kusto
arg("").Resources
| where type =~ "microsoft.hybridcompute/machines"
| extend Company = tostring(tags.Company)
| extend Site = tostring(tags.Site)
| extend ClusterId = tostring(tags.ClusterId)
| extend Env = tostring(tags.Env)
| extend ClusterRg = tostring(tags.Cluster)
| where Company == "CompanyA" and Site == "SiteA" and ClusterId == "C1" and Env == "Prod"
| distinct ClusterRg
```

**Grafana variable queries** in [`grafana/queries/`](../grafana/queries/) use this pattern.

**Notes**

- Preview feature; editor may show false syntax errors — run the query anyway
- `arg()` returns max **1,000 rows** per query (enough for variable dropdowns)
- Tag property names are case-sensitive (`tags.Company`, not `tags.company`)

Reference: [Correlate ARG with Log Analytics](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/azure-monitor-data-explorer-proxy)

---

## Option B — Custom table + scheduled sync (optional DCR)

Use this only if you need a **physical table** in the LAW (offline queries, no ARG dependency in Grafana, or arg() unavailable).

### Architecture

```
Azure Resource Graph
        │
        ▼
Azure Automation runbook (daily/hourly)
        │
        ▼
Logs Ingestion API  ──►  DCR (Direct)  ──►  HybridResourceTags_CL
```

### Step 1 — Create custom table

```bash
az monitor log-analytics workspace table create \
  --resource-group azr-mon-rg-monitoring \
  --workspace-name azr-mon-law-central \
  --name HybridResourceTags_CL \
  --columns '[{"name":"TimeGenerated","type":"datetime"},{"name":"ResourceId","type":"string"},{"name":"Company","type":"string"},{"name":"Site","type":"string"},{"name":"ClusterId","type":"string"},{"name":"Env","type":"string"},{"name":"ClusterRg","type":"string"},{"name":"SubscriptionId","type":"string"},{"name":"Name","type":"string"}]'
```

### Step 2 — Create ingestion DCR (Direct kind)

Deploy [`infra/modules/dcr-resource-tags.bicep`](../infra/modules/dcr-resource-tags.bicep) or create in Portal:

- **Stream**: `Custom-HybridResourceTags`
- **Destination**: central LAW
- **Transform**: `source`
- **Output**: `HybridResourceTags_CL`

This DCR does **not** attach to Arc machines — it receives data from the **Logs Ingestion API** only.

### Step 3 — Grant Automation managed identity

- Role: **Monitoring Metrics Publisher** on the DCR
- Role: **Reader** on subscriptions (to query ARG)

### Step 4 — Runbook posts tag inventory

Runbook queries ARG and POSTs JSON to the DCR immutable ID endpoint (see Microsoft sample for Logs Ingestion API).

### Step 5 — Grafana variables query custom table

```kusto
HybridResourceTags_CL
| where TimeGenerated > ago(1d)
| summarize arg_max(TimeGenerated, *) by ResourceId
| extend Company = Company
| distinct Company
```

---

## Option C — Legacy: HTTP Data Collector (no DCR)

Older pattern: Automation → HTTP Data Collector API → `VMResourceTags_CL`. Microsoft recommends **Logs Ingestion API + DCR** (Option B) for new deployments.

---

## What your Default/Custom DCR already collect

Your AMA DCRs ([`dcr-default.bicep`](../infra/modules/dcr-default.bicep)) collect telemetry only:

- `Microsoft-Perf` → `Perf`
- `Microsoft-Event` → `Event` (includes `_ResourceId` on each row)
- `Microsoft-InsightsMetrics` → `InsightsMetrics`
- Heartbeat (automatic with LAW association)

Each row includes `_ResourceId`, which you can **join** to tag data:

```kusto
Heartbeat
| where TimeGenerated > ago(1h)
| lookup (
    arg("").Resources
    | where type =~ "microsoft.hybridcompute/machines"
    | project _ResourceId=tolower(id), tags
) on _ResourceId
| where tostring(tags.Company) == "CompanyA"
```

---

## Verify tag queries work

Run in central LAW:

```kusto
// Option A — ARG cross-query
arg("").Resources
| where type =~ "microsoft.hybridcompute/machines"
| where isnotempty(tags.Company)
| project name, resourceGroup, subscriptionId, tags
| take 10

// Option B — custom table (if deployed)
HybridResourceTags_CL
| summarize arg_max(TimeGenerated, *) by ResourceId
| take 10
```

---

## Summary

| Goal | Approach |
|------|----------|
| Grafana tag variables (`company`, `site`, …) | **Option A**: `arg("").Resources` — update queries in `grafana/queries/` |
| Physical tag table in LAW | **Option B**: Custom table + ingestion DCR + Automation |
| Agent metrics/logs | Existing **Default/Custom DCR** (unchanged) |

**Do not add tag collection to the Default DCR** — it will not work; AMA cannot read ARM tags.
