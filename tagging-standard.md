# Hybrid Monitoring Tagging Standard

All Arc-enabled servers, Azure VMs, and cluster resource groups that report to the central Log Analytics workspace must carry the tags below. Tags mirror the architecture grouping **Site / Company → Cluster → Physical Node** and drive Azure Policy DCR associations and Grafana dashboard scoping.

## Architecture alignment

The monitoring design centralizes telemetry from multiple sites, companies, and clusters into one Log Analytics workspace via shared Data Collection Rules:

```
Site/Company A ── Cluster C1 ── Physical Node 1 (HCI OS + AMA)
              │              └── Physical Node 2 (HCI OS + AMA)
              └── Cluster C2 ── Physical Node 1 (HCI OS + AMA)
                               └── Physical Node 2 (HCI OS + AMA)

Site/Company B ── Cluster C1 ── Physical Node 1 (HCI OS + AMA)
                               └── Physical Node 2 (HCI OS + AMA)
                              │
                              ▼
              Data Collection Rules (Default + Custom)
                              │
                              ▼
              Azure Log Analytics Workspace (Azure Region 1)
```

Each **physical node** is an Arc-enabled machine (`Microsoft.HybridCompute/machines`) with the Azure Monitor Windows Agent extension. Node identity in queries comes from `Computer` / `HostKey` in Heartbeat and Perf; tags identify **where** the node belongs in the hierarchy.

## Tag hierarchy

| Level | Diagram label | Tag | Example |
|-------|---------------|-----|---------|
| Organization | Site/Company A, Site/Company B | `Company` | `CompanyA`, `CompanyB` |
| Location / tenant scope | Site (within or across companies) | `Site` | `NYC`, `Sydney`, `SiteA` |
| Cluster instance | C1, C2 | `ClusterId` | `C1`, `C2` |
| Azure resource group | (maps to Grafana `${cluster}`) | `Cluster` | `azr-131-itsusra1-delldev01` |
| Physical node | Physical Node 1, Physical Node 2 | *(resource name)* | `itsusra1dazl001` |
| Platform | HCI OS | `Platform` | `HCI` |

**Display grouping** in dashboards and reports should follow: `Company` → `Site` → `ClusterId` → node (`Computer`).

## Required tags

Apply to **Arc machines**, **Azure VMs**, and **cluster resource groups**.

| Tag | Example values | Purpose |
|-----|----------------|---------|
| `Company` | `CompanyA`, `CompanyB` | Tenant / business-unit boundary (matches diagram Site/Company boxes) |
| `Site` | `NYC`, `Sydney`, `SiteA` | Geographic or logical site for grouping and cost/showback |
| `ClusterId` | `C1`, `C2` | Cluster instance within a company/site (matches diagram C1, C2) |
| `Cluster` | `azr-131-itsusra1-delldev01` | Azure resource group name — maps to Grafana `${cluster}` variable |
| `Env` | `Prod`, `Dev`, `Test` | Environment scoping |
| `Platform` | `HCI`, `Windows`, `Linux` | OS / platform type (HCI nodes use `HCI`) |
| `ObservabilityScope` | `hybrid-baseline` | Triggers **Default DCR** policy association |

## Conditional tags

| Tag | When required | Example values | Purpose |
|-----|---------------|----------------|---------|
| `Role` | When Custom DCR collection is needed | `HCI`, `Web`, `SQL`, `App`, `Infra` | Triggers **Custom DCR** policy association |

Machines without a `Role` tag receive only the Default DCR. Machines with `Role=Web|SQL|App|HCI` receive **both** Default and Custom DCR associations.

## Tag examples (per architecture diagram)

### Site/Company A — Cluster C1 (Azure Local HCI, production)

Represents the first cluster box under Company A in the architecture diagram.

```
Company=CompanyA
Site=SiteA
ClusterId=C1
Cluster=azr-131-itsusra1-delldev01
Env=Prod
Platform=HCI
ObservabilityScope=hybrid-baseline
Role=HCI
```

### Site/Company A — Cluster C2

Second cluster under the same company, different resource group.

```
Company=CompanyA
Site=SiteA
ClusterId=C2
Cluster=azr-100-companya-c2-rg
Env=Prod
Platform=HCI
ObservabilityScope=hybrid-baseline
Role=HCI
```

### Site/Company B — Cluster C1

Separate company/tenant, own subscription or region.

```
Company=CompanyB
Site=SiteB
ClusterId=C1
Cluster=azr-200-companyb-c1-rg
Env=Prod
Platform=HCI
ObservabilityScope=hybrid-baseline
Role=HCI
```

### Physical node (Arc machine)

Tags are set on the Arc machine resource; node name is the machine name (e.g. `itsusra1dazl001`), not a tag.

```
Company=CompanyA
Site=SiteA
ClusterId=C1
Cluster=azr-131-itsusra1-delldev01
Env=Prod
Platform=HCI
ObservabilityScope=hybrid-baseline
Role=HCI
```

### IIS web server (Custom DCR)

```
Company=CompanyA
Site=NYC
ClusterId=C1
Cluster=azr-200-web-rg
Env=Prod
Platform=Windows
ObservabilityScope=hybrid-baseline
Role=Web
```

### Dev/test cluster

```
Company=CompanyA
Site=SiteA
ClusterId=C1
Cluster=azr-dev-cluster-rg
Env=Test
Platform=HCI
ObservabilityScope=hybrid-baseline
```

## Site-to-tag mapping (shared DCR model)

All sites and companies use the same shared DCRs in the monitoring subscription. Differences are expressed through tags, not separate DCRs.

| Diagram group | Company | Site | ClusterId | Cluster RG (example) |
|---------------|---------|------|-----------|----------------------|
| Site/Company A C1 | `CompanyA` | `SiteA` | `C1` | `azr-131-itsusra1-delldev01` |
| Site/Company A C2 | `CompanyA` | `SiteA` | `C2` | `azr-100-companya-c2-rg` |
| Site/Company B C1 | `CompanyB` | `SiteB` | `C1` | `azr-200-companyb-c1-rg` |

## Apply tags

Use the tagging script for single resources or bulk Arc machines in a cluster RG:

```bash
# Bulk tag all Arc machines + RG in a cluster (Company A / C1 POC)
SUBSCRIPTION_ID=<sub-id> \
RESOURCE_GROUP=azr-131-itsusra1-delldev01 \
COMPANY=CompanyA \
SITE=SiteA \
CLUSTER_ID=C1 \
ENV=Prod \
PLATFORM=HCI \
ROLE=HCI \
CLUSTER=azr-131-itsusra1-delldev01 \
./scripts/apply-tags.sh --bulk-arc

# Single Arc machine (physical node)
SUBSCRIPTION_ID=<sub-id> \
RESOURCE_GROUP=azr-131-itsusra1-delldev01 \
RESOURCE_NAME=itsusra1dazl001 \
COMPANY=CompanyA \
SITE=SiteA \
CLUSTER_ID=C1 \
ENV=Prod \
PLATFORM=HCI \
ROLE=HCI \
CLUSTER=azr-131-itsusra1-delldev01 \
./scripts/apply-tags.sh
```

Or via Azure CLI:

```bash
az tag update \
  --resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.HybridCompute/machines/<name> \
  --tags \
    Company=CompanyA \
    Site=SiteA \
    ClusterId=C1 \
    Cluster=azr-131-itsusra1-delldev01 \
    Env=Prod \
    Platform=HCI \
    ObservabilityScope=hybrid-baseline \
    Role=HCI
```

Tag the **cluster resource group** with the same `Company`, `Site`, `ClusterId`, `Cluster`, `Env`, and `Platform` values so RG-level resources inherit consistent scope.

## Grafana and KQL scoping

Full variable definitions: [grafana-variables.md](grafana-variables.md)

| Order | Grafana variable | Tag / source | Notes |
|-------|------------------|--------------|-------|
| 1 | `az_monitor` | Data source | Central LAW |
| 2 | `law_subscription` | Monitoring sub (hidden) | LAW hosting subscription for `AzureResource` queries |
| 3 | `company` | `tags.Company` | Include All |
| 4 | `site` | `tags.Site` | Filtered by `${company}` |
| 5 | `cluster_id` | `tags.ClusterId` | Filtered by `${company}`, `${site}` |
| 6 | `env` | `tags.Env` | Filtered by hierarchy above |
| 7 | `cluster` | `tags.Cluster` (RG name) | Drives panel resource scope |
| 8 | `subscription` | `subscriptionId` from Arc machine | Site subscription for `${cluster}` |
| 9 | `computer` | SDDC 3000 + Heartbeat | Physical node — `grafana/queries/list_computer_query.kql` |
| 10 | `disk` / `pool` | SDDC 3002 | Volume/pool panels |

Example KQL filter by hierarchy (when tags are available on `AzureResource` or custom columns):

```kusto
| where tostring(Tags.Company) == "CompanyA"
| where tostring(Tags.Site) == "SiteA"
| where tostring(Tags.ClusterId) == "C1"
```

Cluster-scoped panels (current AZR-208 pattern):

```kusto
| where _ResourceId has tolower("/resourcegroups/${cluster}/")
```

## Policy behavior

| Tag state | Default DCR | Custom DCR |
|-----------|-------------|------------|
| `ObservabilityScope=hybrid-baseline` | Associated | — |
| `Role=Web` (or SQL/App/HCI) | Associated (if ObservabilityScope set) | Associated |
| Missing `ObservabilityScope` | Not associated by tag policy | — |

After tagging, allow up to 30 minutes for Azure Policy remediation. Trigger on-demand remediation from the Policy compliance blade if needed.

## Naming alignment

| Concept | Convention | Example |
|---------|------------|---------|
| Diagram group | `Site/Company {Company} {ClusterId}` | Site/Company A C1 |
| Company tag | `Company{A\|B\|...}` | `CompanyA` |
| ClusterId tag | `C1`, `C2`, ... | `C1` |
| Cluster RG | `azr-{sub}-{site}{cluster}-{name}` | `azr-131-itsusra1-delldev01` |
| Arc machine (node) | hostname | `itsusra1dazl001` |
| Default DCR | `dcr-hybrid-default-shared` | Shared across all sites/companies |
| Custom DCR | `dcr-hybrid-custom-shared` | Shared across all sites/companies |
| Central LAW | `azr-mon-law-central` | Monitoring subscription, Azure Region 1 |
