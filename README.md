# ms-alz-bicep

Custom Cloud Adoption Framework (CAF) aligned Azure landing zone in Bicep, built on [Azure Verified Modules (AVM)](https://aka.ms/avm), extended with an AI Landing Zone workload pattern.

> **Status:** Scaffold only. Folder structure and READMEs are in place; no Bicep templates, parameters or pipelines have been implemented yet.

## Architecture

### Management group hierarchy

```
Tenant Root Group
└── <prefix> (intermediate root)
    ├── Platform
    │   ├── Management        Log Analytics, monitoring, Sentinel
    │   ├── Connectivity      Hub / vWAN, firewall, private DNS
    │   └── Identity          Domain controllers / Entra DS (optional)
    ├── Landing Zones
    │   ├── Corp              Private, hub-connected workloads (incl. AI landing zone)
    │   └── Online            Internet-facing workloads
    ├── Sandbox               Unrestricted experimentation, no hub connectivity
    └── Decommissioned        Subscriptions pending cancellation
```

### Layers

| Layer | Folder | Purpose |
|---|---|---|
| Governance | `platform/00-governance` | Management groups, policy definitions/assignments, RBAC |
| Management | `platform/10-management` | Central logging, monitoring, security operations |
| Connectivity | `platform/20-connectivity` | Hub networking, firewall policy, central private DNS zones |
| Identity | `platform/30-identity` | Identity subscription resources |
| Landing zones | `landing-zones` | Subscription vending: subscription, MG placement, spoke VNet, peering, budgets, RBAC |
| Workloads | `workloads` | Application deployments into vended subscriptions |

### AI Landing Zone

The AI Landing Zone (`workloads/ai-landing-zone`) is deployed as a **workload** into a vended subscription under `Landing Zones/Corp` (or `Online`), aligned to the Azure AI Landing Zones reference architecture. It consumes platform services rather than duplicating them:

| From the platform | Used by the AI landing zone for |
|---|---|
| Hub network / firewall | Spoke peering, egress control |
| Central private DNS zones | Private endpoints for AI Foundry, OpenAI, AI Search, Storage, Key Vault |
| Log Analytics workspace | Diagnostic settings, Application Insights |
| Azure Policy | AI guardrails enforced at the Landing Zones MG |

**AI guardrail policies** (in `platform/00-governance/policy/definitions`):

- Deny public network access on AI services
- Require private endpoints
- Disable local authentication (API keys); use Entra ID / managed identity
- Restrict to approved model deployments, SKUs and regions
- Require diagnostic settings to the central workspace

Components shared by multiple AI workloads (e.g. APIM AI gateway) should be promoted to a `platform/40-ai-shared/` layer.

## Repository structure

```
.github/workflows/              Pipelines, one per layer
config/                         Environment-specific values
  dev/  prod/
modules/                        Shared wrappers around AVM
  naming/  tags/  custom/
platform/                       CAF platform layer
  00-governance/
    management-groups/
    policy/definitions/
    policy/assignments/
    rbac/
  10-management/
  20-connectivity/
    hub-and-spoke/              (or vwan/)
    private-dns/
    firewall-policy/
  30-identity/
landing-zones/
  vending/                      Subscription vending module
  subscriptions/                One .bicepparam per landing zone
workloads/
  _templates/                   Starter for new workloads
  ai-landing-zone/
    modules/
      foundry/  openai/  search/  data/
      apim-ai-gateway/  compute/  security/  monitoring/
    params/                     dev / prod .bicepparam
tests/                          Lint, what-if, PSRule for Azure
docs/
  architecture/                 ADRs, diagrams
  runbooks/
```

Each folder contains a `README.md` describing its intended contents.

## Deployment order

| Step | Folder | Scope |
|---|---|---|
| 1 | `platform/00-governance` | Tenant / management group |
| 2 | `platform/10-management` | Management subscription |
| 3 | `platform/20-connectivity` | Connectivity subscription |
| 4 | `platform/30-identity` | Identity subscription |
| 5 | `landing-zones/vending` | Billing scope / management group |
| 6 | `workloads/*` | Workload subscription |

## Prerequisites

- Azure CLI and Bicep CLI (latest)
- Entra ID tenant with permissions to create management groups and assign policy at the intermediate root
- Billing scope (EA / MCA) with rights to create subscriptions for vending
- Platform subscriptions: Management, Connectivity, Identity (optional)
- Deployment identities (workload identity federation) per pipeline

## Principles

- **Logic vs. configuration:** logic in `.bicep`, values in `.bicepparam`. No environment values hard-coded in templates.
- **AVM first:** reference `br/public:avm/...` directly; wrap in `modules/` only to apply org defaults.
- **Consume, don't duplicate:** workloads use platform networking, DNS, logging and policy.
- **Least privilege:** one pipeline and deployment identity per layer.
- **Validate before deploy:** lint, `what-if` and PSRule for Azure run on every pull request.

## References

- [Cloud Adoption Framework – Azure landing zones](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- [Azure Verified Modules](https://aka.ms/avm)
- [ALZ Bicep](https://github.com/Azure/ALZ-Bicep)
- [Azure AI Landing Zones](https://github.com/Azure/AI-Landing-Zones)
- [Subscription vending](https://learn.microsoft.com/azure/architecture/landing-zones/subscription-vending)
