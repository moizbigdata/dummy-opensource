# Grafana RBAC for Hybrid Monitoring

Self-hosted Grafana uses the Azure Monitor data source plugin to query the central Log Analytics workspace and list ARM resources for dashboard variables.

## Service principal requirements

Grant the Grafana service principal these roles:

| Role | Scope | Purpose |
|------|-------|---------|
| **Log Analytics Reader** | Central LAW | Run KQL queries (Perf, Event, Heartbeat, InsightsMetrics) |
| **Reader** | Site subscriptions and/or cluster RGs | ARM-based variables (`microsoft.hybridcompute/machines`) |
| **Monitoring Reader** | Optional — cluster RGs | Azure Stack HCI platform metrics (`microsoft.azurestackhci/clusters`) |

### POC service principal

Client ID (from existing POC): `1dbeedd6-ac59-4d5c-ae41-4850755c7670`

## Assign roles (Cloud Ops)

```bash
GRAFANA_SP_OBJECT_ID="<grafana-sp-object-id>"
LAW_ID="/subscriptions/<mon-sub>/resourceGroups/azr-mon-rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/azr-mon-law-central"
SITE_SUB_ID="<site-subscription-id>"
CLUSTER_RG="azr-131-itsusra1-delldev01"

# Log Analytics Reader on central workspace
az role assignment create \
  --assignee-object-id "${GRAFANA_SP_OBJECT_ID}" \
  --assignee-principal-type ServicePrincipal \
  --role "Log Analytics Reader" \
  --scope "${LAW_ID}"

# Reader on site subscription (ARM variables)
az role assignment create \
  --assignee-object-id "${GRAFANA_SP_OBJECT_ID}" \
  --assignee-principal-type ServicePrincipal \
  --role "Reader" \
  --scope "/subscriptions/${SITE_SUB_ID}"

# Optional: Reader scoped to cluster RG only (least privilege)
az role assignment create \
  --assignee-object-id "${GRAFANA_SP_OBJECT_ID}" \
  --assignee-principal-type ServicePrincipal \
  --role "Reader" \
  --scope "/subscriptions/${SITE_SUB_ID}/resourceGroups/${CLUSTER_RG}"
```

## Grafana data source configuration

| Setting | Value |
|---------|-------|
| Authentication | App Registration / Service Principal |
| Tenant ID | Your Azure AD tenant |
| Client ID | Grafana SP client ID |
| Default subscription | Site subscription for multi-cluster dashboards |
| Log Analytics default workspace | Central LAW (`azr-mon-law-central`) |

## Dashboard variables (AZR-208)

| Variable | Source | RBAC needed |
|----------|--------|-------------|
| `az_monitor` | Azure Monitor datasource | LAW access |
| `subscription` | Azure Subscriptions API | Reader on subscription |
| `cluster` | Resource Groups | Reader on subscription |
| `computer` | **KQL** ([`list_computer_query`](../list_computer_query)) | Log Analytics Reader only |
| `disk` / `pool` | SDDC EventID 3002 KQL | Log Analytics Reader only |

### Recommended: SDDC KQL for `computer` variable

For Azure Local HCI, prefer the SDDC EventID 3000 + Heartbeat KQL variable over ARM `microsoft.hybridcompute/machines`:

- No extra ARM Reader permission required
- Lists only HCI cluster nodes (not unrelated Arc machines)
- Matches panel query hostname logic (`HostKey` join)

See [`list_computer_variable_arm.md`](../list_computer_variable_arm.md) for ARM troubleshooting if HybridCompute namespace is missing from the dropdown.

## Multi-site dashboard patterns

**Per-cluster dashboard (current AZR-208):**

- Resource scope: `/subscriptions/${subscription}/resourcegroups/${cluster}`
- KQL filter: `| where _ResourceId has tolower("/resourcegroups/${cluster}/")`

**Cross-site overview (future):**

- Query central LAW without cluster filter
- Add `Site` tag column if propagated to custom logs
- Or use subscription variable to switch sites

## Verification

1. Open Grafana → Explore → Azure Monitor → Logs
2. Run: `Heartbeat | where TimeGenerated > ago(1h) | take 10`
3. Open AZR-208 dashboard; confirm `computer` variable populates
4. If ARM variable needed: confirm `microsoft.hybridcompute` appears after Reader is granted
