# Hybrid Monitoring Onboarding Runbook

Step-by-step guide to onboard Azure Arc servers, Azure VMs, and Azure Local HCI clusters from multiple sites/subscriptions into the central Log Analytics workspace.

## Prerequisites

- Monitoring subscription with Contributor access
- Cloud Ops: network/private endpoint access to LAW (if applicable)
- Arc agents installed on on-prem/HCI nodes
- Azure CLI 2.50+ with `az bicep` available

## Phase 1 — Platform foundation (monitoring subscription)

### 1.1 Deploy central LAW and shared DCRs

```bash
# Production monitoring subscription
SUBSCRIPTION_ID=<monitoring-sub-id> \
RESOURCE_GROUP=azr-mon-rg-monitoring \
PARAM_FILE=infra/parameters/monitoring.prod.bicepparam \
./scripts/deploy.sh
```

Record outputs:

- `logAnalyticsWorkspaceId`
- `defaultDcrId`
- `customDcrId`

### 1.2 Verify deployment

```bash
az monitor data-collection rule show \
  --name dcr-hybrid-default-shared \
  --resource-group azr-mon-rg-monitoring

az monitor data-collection rule show \
  --name dcr-hybrid-custom-shared \
  --resource-group azr-mon-rg-monitoring
```

### 1.3 Policy assignments

Policy assignments are deployed automatically when `deployPolicies=true`. Verify in Azure Portal:

- **Hybrid monitoring — associate Default DCR by ObservabilityScope tag**
- **Hybrid monitoring — associate Custom DCR by Role tag**
- **Hybrid monitoring — deploy AMA on Arc Windows machines**

Assign policies at management group level for all site subscriptions, or per subscription during phased rollout.

## Phase 2 — Per-site / per-cluster onboarding

### 2.1 Arc onboarding

Ensure each HCI node and standalone server is registered:

```bash
az connectedmachine list \
  --resource-group <cluster-rg> \
  --output table
```

### 2.2 Apply tags

See [tagging-standard.md](tagging-standard.md). Minimum for baseline monitoring:

```
Site=<site>
Env=Prod|Dev|Test
ObservabilityScope=hybrid-baseline
Cluster=<cluster-rg-name>
Role=HCI   # optional; adds Custom DCR for storage/IIS extras
```

```bash
SUBSCRIPTION_ID=<site-sub-id> \
RESOURCE_GROUP=azr-131-itsusra1-delldev01 \
SITE=POC ENV=Prod ROLE=HCI \
CLUSTER=azr-131-itsusra1-delldev01 \
./scripts/apply-tags.sh --bulk-arc
```

### 2.3 Install Azure Monitor Agent

If not deployed by policy:

```bash
az connectedmachine extension create \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --type AzureMonitorWindowsAgent \
  --resource-group <cluster-rg> \
  --machine-name <arc-machine-name>
```

### 2.4 Verify DCR associations

```bash
az monitor data-collection rule association list \
  --resource /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.HybridCompute/machines/<name> \
  --output table
```

Expected: associations to both `dcr-hybrid-default-shared` and `dcr-hybrid-custom-shared` (if `Role` tag set).

### 2.5 Register Azure Local cluster (optional platform metrics)

```bash
az resource show \
  --resource-group <cluster-rg> \
  --resource-type Microsoft.AzureStackHCI/clusters \
  --name <cluster-name>
```

## Phase 3 — Validation

### 3.1 Run automated checks

```bash
SUBSCRIPTION_ID=<monitoring-sub-id> \
RESOURCE_GROUP=azr-mon-rg-monitoring \
WORKSPACE_ID=azr-mon-law-central \
CLUSTER_RG=azr-131-itsusra1-delldev01 \
./scripts/pilot-validation.sh
```

### 3.2 Run KQL validation pack

Open [`validation/onboarding-checks.kql`](../validation/onboarding-checks.kql) in Log Analytics. Replace `ClusterRg` with your cluster resource group.

**Pass criteria:**

| Check | Expected |
|-------|----------|
| Heartbeat | All HCI nodes appear with `Category=Azure Monitor Agent` |
| SDDC EventID 3000 | Recent inventory events per cluster |
| SDDC EventID 3002 | Volume/pool events for storage panels |
| Perf counters | `Processor Information`, `Memory`, `LogicalDisk`, `Network Interface`, `System` |
| InsightsMetrics | Present as fallback on Arc hosts |

### 3.3 Validate Grafana AZR-208 dashboard

1. Set datasource `${az_monitor}` to central LAW
2. Set `${subscription}` to site subscription
3. Set `${cluster}` to cluster RG name
4. Confirm panels: Server Stats, CPU, Memory, Disk, Network, Volume Usage, Node Up/Down

Use [`list_computer_query`](../list_computer_query) for the `computer` variable (SDDC-based, no ARM Reader required).

## Phase 4 — Migrate from legacy per-cluster DCR

For POC cluster using `azr-131-dcr-delldev01`:

1. Deploy shared `dcr-hybrid-default-shared` (Phase 1)
2. Tag all Arc machines with `ObservabilityScope=hybrid-baseline`
3. Wait for policy to create new association
4. Remove old association to `azr-131-dcr-delldev01`
5. Run validation; confirm AZR-208 panels unchanged
6. Decommission legacy DCR after 7-day parallel run

## Phase 5 — Multi-site expansion

Repeat Phase 2–3 for each site subscription:

| Site | Subscription | Cluster RG example |
|------|-------------|-------------------|
| NYC | Sub-A | `azr-100-nyc-cluster01` |
| Sydney | Sub-B | `azr-200-syd-cluster01` |
| Dev/Test | Sub-A/B | `azr-dev-cluster01` |

Use [`scripts/rollout-site.sh`](../scripts/rollout-site.sh) for a repeatable site checklist.

## Troubleshooting

| Symptom | Action |
|---------|--------|
| No Heartbeat | Verify AMA extension; check DCR association; confirm network egress |
| SDDC events missing | Confirm `Microsoft-Windows-SDDC-Management/Operational` in Default DCR |
| Perf counters partial on Arc | Expected — use explicit counters + InsightsMetrics; see `server_stat_test` |
| Policy not associating DCR | Verify `ObservabilityScope=hybrid-baseline` tag; trigger remediation |
| Grafana computer list empty | Use SDDC KQL variable; or grant ARM Reader (see grafana-rbac.md) |
| Hostname mismatch in panels | Queries use `HostKey = toupper(split(Computer,".")[0])` — no change needed |

## Related files

- Infrastructure: [`infra/main.bicep`](../infra/main.bicep)
- Default DCR spec: [`infra/modules/dcr-default.bicep`](../infra/modules/dcr-default.bicep)
- Custom DCR spec: [`infra/modules/dcr-custom.bicep`](../infra/modules/dcr-custom.bicep)
- Tagging: [tagging-standard.md](tagging-standard.md)
- Grafana RBAC: [grafana-rbac.md](grafana-rbac.md)
