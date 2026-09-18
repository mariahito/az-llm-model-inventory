# Get-AzModelInventory

A PowerShell script that inventories AI model deployments across your Azure subscriptions, enriches each deployment with **lifecycle and retirement data** from the Cognitive Services Models API, and exports a color-coded report to CSV.

## Overview

This script scans all enabled Azure subscriptions accessible to the signed-in account and produces a consolidated CSV report covering:

- **Microsoft Foundry** (`AIServices`) deployments from all catalog publishers, including OpenAI, Anthropic, Meta, Mistral AI, Cohere, DeepSeek, Microsoft, xAI, and others
- **Azure OpenAI** (`OpenAI`) Cognitive Services deployments
- **Azure Machine Learning** online endpoint model deployments

For every Microsoft Foundry and Azure OpenAI deployment, the script calls the Cognitive Services Models API to determine the model's current lifecycle status and published inference retirement date. Publisher, model name, and version are matched together so similarly named models from different publishers cannot collide. Deployments are then classified into a **RetirementRisk** tier so you can immediately see which models need attention.

> **Scope:** The report inventories deployed models, not every model available in the Foundry catalog. Legacy Azure Machine Learning `serverlessEndpoints` resources are not currently scanned.

## Retirement risk tiers

| Console color | Risk tier | Meaning |
|---|---|---|
| 🔴 Red | `Retired` | Model is no longer listed by Azure — deployments will fail |
| 🔴 Red | `Critical` | Retirement date is within **30 days** |
| 🟡 Yellow | `High` | Retirement date is within **31–60 days** |
| 🔵 Cyan | `Medium` | Retirement date is within **61–90 days** |
| 🟢 Green | `Low` | Retirement date is more than **90 days** away |
| ⚪ White | `None` | No retirement date published, or AML endpoint (no data) |
| 🟡 Yellow | `Unknown` | Lifecycle lookup failed; manual review is required |

## CSV columns

| Column | Description |
|---|---|
| `Source` | `MicrosoftFoundry`, `AzureOpenAI`, or `AMLOnlineEndpoint` |
| `SubscriptionId` | Azure subscription ID |
| `ResourceGroup` | Resource group name |
| `AccountName` | Microsoft Foundry/Azure OpenAI account or AML endpoint name |
| `Location` | Azure region |
| `DeploymentName` | Name of the model deployment |
| `ModelPublisher` | Foundry catalog publisher/format (for example, `OpenAI`, `Meta`, `Anthropic`, or `Mistral AI`) |
| `ModelName` | Model identifier (e.g. `gpt-4o`, `gpt-4.1`) |
| `ModelVersion` | Deployed model version |
| `SkuName` | SKU name (e.g. `Standard`, `GlobalStandard`) |
| `Capacity` | Provisioned capacity (PTUs or TPM units) |
| `UpgradePolicy` | Auto-upgrade policy for the deployment |
| `ProvisionState` | Provisioning state (e.g. `Succeeded`, `Failed`, `Disabled`) |
| `LifecycleStatus` | `GenerallyAvailable`, `Preview`, `Deprecating`, `Legacy`, `Retired`, or `Unknown` |
| `RetirementDate` | Published retirement date in `yyyy-MM-dd` format, or empty |
| `DaysUntilRetirement` | Integer days until retirement; negative means already past the date |
| `RetirementRisk` | `Retired`, `Critical`, `High`, `Medium`, `Low`, `None`, or `Unknown` |
| `ResourceId` | Full Azure resource ID |

## Prerequisites

| Requirement | Notes |
|---|---|
| Azure CLI | [Install guide](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| Azure CLI login | Run `az login` before executing the script |
| PowerShell 5.1+ | PowerShell 7 recommended |
| Reader role | Required on all target subscriptions |
| `resource-graph` extension | **Auto-installed** by the script if missing |

## Usage

```powershell
# Scan all enabled subscriptions (default)
.\Get-AzModelInventory.ps1

# Save report to a specific path
.\Get-AzModelInventory.ps1 -OutputPath "C:\Reports\model-inventory.csv"

# Scan specific subscriptions only
.\Get-AzModelInventory.ps1 -SubscriptionIds "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx", "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"

# Combine both parameters
.\Get-AzModelInventory.ps1 -SubscriptionIds "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" -OutputPath "C:\Reports\model-inventory.csv"
```

## Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `-OutputPath` | `string` | No | `model-inventory-<timestamp>.csv` in current directory | Path to the output CSV file |
| `-SubscriptionIds` | `string[]` | No | All enabled subscriptions | One or more subscription IDs to scan |

## Sample output

```
Checking prerequisites...
  Discovered 12 enabled subscription(s).

[1/4] Querying Microsoft Foundry and Azure OpenAI accounts via Resource Graph...
      Found 4 Microsoft Foundry/Azure OpenAI account(s).
[2/4] Retrieving model deployments and lifecycle data from each Microsoft Foundry/Azure OpenAI account...
      Found 2 Microsoft Foundry deployment(s).
      Found 3 Azure OpenAI deployment(s).
[3/4] Querying Azure ML online endpoints via Resource Graph...
      Found 0 AML online endpoint(s).
[4/4] Retrieving model deployments from each AML online endpoint...
      Found 0 AML deployment(s).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Scan complete.
  Subscriptions scanned : 12
  Foundry deploys       : 2
  Azure OpenAI deploys  : 3
  AML endpoint deploys  : 0
  Total deployments     : 5

  Deployed model publishers:
    Meta : 1
    Mistral AI : 1
    OpenAI : 3

  Retirement Risk Summary:
    [RETIRED ]  1 deployment(s) — model no longer available, immediate action required!
    [CRITICAL]  1 deployment(s) — retiring within 30 days!
    [HIGH    ]  1 deployment(s) — retiring within 31-60 days
    [LOW     ]  2 deployment(s) — retiring in more than 90 days

  Report saved to       : C:\Reports\model-inventory-20260429-090000.csv
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Risk       Source           ResourceGroup        AccountName            DeploymentName         Publisher        ModelName              Version        RetirementDate Days    ProvisionState
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Critical   AzureOpenAI      rg-prod              oai-prod               gpt4-old               OpenAI           gpt-4                  0613           2026-05-15     17      Succeeded
High       MicrosoftFoundry rg-ai                foundry-prod           llama-prod             Meta             Llama-3.3-70B-Instruct 1              2026-06-10     43      Succeeded
Low        MicrosoftFoundry rg-ai                foundry-prod           mistral-prod            Mistral AI       Mistral-Large          1              2026-09-30     155     Succeeded
```

## How retirement is detected

The script calls the **Cognitive Services Models API** (`GET .../models?api-version=2024-10-01`) once per Microsoft Foundry or Azure OpenAI account and caches the result. This API returns `lifecycleStatus` and the inference retirement date in `deprecation.inference` for model versions available from each publisher in that account's region.

| Scenario | Result |
|---|---|
| Publisher, model, and version present with `deprecation.inference` | `RetirementDate` and `DaysUntilRetirement` are populated; risk tier is calculated |
| Model present, no inference retirement date | Lifecycle status is retained; risk is `None` |
| Model **not present** in the API response | Treated as `Retired` — Azure removes retired models from the listing entirely |
| Models API lookup fails | Lifecycle and retirement risk are `Unknown`, requiring manual review |

> **Note:** Microsoft publishes retirement dates in advance via the [Azure OpenAI model retirements documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/model-retirements). The `RetirementDate` column in the CSV is sourced directly from the ARM API and will be empty when Microsoft has not yet announced a date.

## Who is this for?

- **Azure administrators / cloud ops teams** — audit what AI models are deployed across a large tenant and spot retirement risks immediately
- **FinOps / cost teams** — identify all deployments and their capacity for cost tracking
- **Security & compliance officers** — verify only approved, non-retired models are deployed
- **AI/ML platform teams** — plan model upgrades proactively before retirement deadlines
- **Microsoft CSAs and partners** — run assessments on behalf of customers with a single command

## How it works

1. **Prerequisites check** — validates Azure CLI login and auto-installs the `resource-graph` extension if needed
2. **Subscription discovery** — enumerates all enabled subscriptions (or uses the list you provide)
3. **Foundry and Azure OpenAI accounts** — queries Azure Resource Graph for Cognitive Services accounts of kind `AIServices` or `OpenAI`
4. **Deployments + lifecycle** — calls the ARM Deployments and Models APIs (`2024-10-01`) per account, matching publisher + model + version; results are cached per account
5. **Retirement classification** — each deployment is assigned a `RetirementRisk` tier based on its retirement date relative to today
6. **AML online endpoints** — queries Azure Resource Graph for all `machinelearningservices/workspaces/onlineendpoints`
7. **AML deployments** — calls the ARM Online Deployments API (`2024-04-01`) for each endpoint
8. **Export** — writes a sorted CSV with all columns and prints a color-coded, urgency-sorted table plus a risk summary banner to the console

Resource Graph queries use automatic pagination (1 000 records per page) and group subscriptions by Entra tenant. The script switches Azure CLI context for each tenant and restores the original subscription when complete.

## Permissions

The signed-in account needs at minimum the **Reader** role on each subscription being scanned. No write permissions are required. The script performs read-only API calls only.

## License

MIT
