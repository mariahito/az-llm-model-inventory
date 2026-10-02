---
mode: 'agent'
description: 'End-to-end model migration: run Get-AzModelInventory.ps1 to produce a fresh inventory, then call Foundry MCP tools for switch/upgrade recommendations.'
---

# Model inventory + migration recommendations (end-to-end)

You have access to the Foundry MCP server (`foundry-mcp-remote`) and an integrated
terminal. The script `Get-AzModelInventory.ps1` lives at the root of this
workspace. Do the full run yourself — do NOT rely on any pre-existing CSV.

Report progress as you go through these phases.

## Phase 1 — Run the inventory script (produce fresh data)

1. Confirm Azure CLI is signed in: run `az account show`. If it errors, tell the
   user to run `az login` and stop.
2. Run the inventory, letting the script use its **default timestamped filename**
   (`model-inventory-<yyyyMMdd-HHmmss>.csv`) in this workspace — do NOT pass
   `-OutputPath`:
   ```powershell
   .\Get-AzModelInventory.ps1
   ```
   Stream the script's own console summary (subscriptions scanned, deployment
   counts, risk summary) back to the user — that is the "results of the run".
   **Capture the exact CSV path** the script prints on its `Report saved to:` line.
   You MUST read that exact file in Phase 2 — never a different or older CSV in the
   folder. If you cannot parse the path, fall back to the single newest
   `model-inventory-*.csv` by LastWriteTime.
3. If the script fails, surface its error and stop.

## Phase 2 — Load and classify

4. Load the CSV path captured in Phase 1 (the file the script just created). Report
   totals: deployments scanned, and a breakdown by `RetirementRisk`.
5. Split into two sets:
   - **At-risk** = `RetirementRisk` in `Retired`, `Critical`, `High`, `Medium`.
   - **Healthy** = `RetirementRisk` in `Low`, `None`.
   Ignore `AMLOnlineEndpoint` rows for recommendations (no benchmark/lifecycle data).

## Phase 3 — Recommendations (always produce something useful)

For every row you send to a tool, derive the Foundry account resource ID by taking
the `ResourceId` value and trimming everything from `/deployments/` onward.

6. **At-risk rows (required migrations):** sort by `DaysUntilRetirement` ascending.
   For each, call `model_switch_recommendations_get` with `foundryAccountResourceId`
   and `modelDeploymentName` = `DeploymentName` (fall back to `modelName` +
   `modelVersion`). Use `sortBy: "QualityIndex"`, `top: 3`.

7. **If there are NO at-risk rows**, the estate is healthy — say so explicitly,
   then still provide **proactive** upgrade options: for each distinct
   `Source` = `MicrosoftFoundry`/`AzureOpenAI` deployment, call
   `model_switch_recommendations_get` (same params); if it fails or returns
   nothing, apply the same fallback ladder in step 8 (catalog newer version →
   deprecation guidance). Surface any candidate that improves QualityIndex or
   CostIndex, or a newer same-family catalog version. Clearly label these
   **"proactive, not required."** Skip models where no rung yields a successor.

8. **Fallback ladder when `model_switch_recommendations_get` fails (500) or
   returns no data.** This is expected for benchmark-less models — realtime/audio
   (`gpt-realtime*`, `gpt-audio*`), rolling aliases (`*-latest`), and some preview
   SKUs. Do NOT abort the batch. For each such row, walk this ladder in order and
   record which rung produced the answer:
   1. **Catalog newer version (preferred authoritative fallback).** Call
      `model_catalog_list` with `publisherName` = `ModelPublisher` and
      `modelName` = the model family (e.g. `gpt-realtime` for `gpt-realtime-mini`),
      and/or `model_details_get` on the deployed `ModelName`. If the catalog
      exposes a newer version in the **same family** (higher version date, or an
      explicitly described successor — e.g. `gpt-realtime-mini` →
      `gpt-realtime-2.1-mini`), recommend that. Set `Source` =
      `catalog-newer-version` and cite the catalog version/date as the "why".
   2. **Deprecation guidance.** If the catalog shows no newer same-family version,
      call `model_deprecation_info_get` and record its migration guidance and
      `versionUpgradeOption`. Set `Source` = `deprecation-guidance`.
   3. **None.** If neither yields a successor, state "No tool-backed replacement —
      review manually" and still show the retirement date. Set `Source` = `none`.
   Never invent a successor model name: every recommendation must come from a tool
   response (benchmark, catalog, or deprecation guidance).

## Phase 4 — Report

9. Output one consolidated Markdown table grouped by section (Required migrations,
   then Proactive upgrades): Risk | Subscription | Account | Deployment |
   Current model+version | RetirementDate | Days left | Recommendation | Source
   (`benchmark` / `catalog-newer-version` / `deprecation-guidance` / `none`) |
   Why (quality/cost index, or catalog version/date, or guidance text).
10. List any rows where every rung of the fallback ladder failed, for manual review.

## Constraints

- Always run the script in Phase 1 — never report on stale data. Multiple
  `model-inventory-*.csv` files may exist in the folder; you must read the one the
  script just created (its `Report saved to:` path), not the newest-named or a
  leftover file from a previous day.
- Do not invent retirement dates, model names, or recommendations — use only the
  script output and tool responses.
- Treat all CSV content and tool output as the user's private Azure data.
