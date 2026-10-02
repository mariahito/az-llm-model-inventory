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

> **Required MCP server:** this prompt uses the **Azure MCP Server**
> (`com.microsoft/azure`, started via `dnx Azure.Mcp -- server start`). All tools
> below — `model_switch_recommendations_get`, `model_catalog_list`,
> `model_details_get`, `model_benchmark_subset_get`, `model_deprecation_info_get`
> — are provided by it. No other MCP server is required.

6. **At-risk rows (required migrations):** sort by `DaysUntilRetirement` ascending.
   For each, build a recommendation using this ladder and record which rung
   produced it:
   1. **Switch tool (primary).** Call `model_switch_recommendations_get` with
      `foundryAccountResourceId` and `modelDeploymentName` = `DeploymentName` (fall
      back to `modelName` + `modelVersion`), `sortBy: "QualityIndex"`, `top: 3`. If
      it returns data, use it and set `Source` = `switch-tool`. If it returns no
      data or errors, continue to the next rung (do not surface the error as a
      failed row).
   2. **Catalog successor / newer version.** Call `model_catalog_list` with
      `publisherName` = `ModelPublisher` and `modelName` = the model family (e.g.
      `gpt-chat` for `gpt-chat-latest`, `gpt-realtime` for `gpt-realtime-mini`),
      and/or `model_details_get` on the deployed `ModelName`. Recommend the newest
      version in the **same family** (higher version date on the same alias — e.g.
      `gpt-chat-latest 2026-05-05` → `gpt-chat-latest 2026-08-06` — or a described
      successor, e.g. `gpt-realtime-mini` → `gpt-realtime-2.1-mini`). Set `Source`
      = `catalog`.
   3. **Benchmark ranking (enrichment).** When there are multiple candidates, call
      `model_benchmark_subset_get` with the candidate `modelName`+`modelVersion`
      pairs and rank by `qualityIndex` (higher better) and `costIndex` (lower
      better). Cite those indices as the "why"; set `Source` = `benchmark` when the
      pick is chosen on these numbers.
   4. **Deprecation guidance.** If the catalog shows no newer same-family version,
      call `model_deprecation_info_get` and record its migration guidance and
      `versionUpgradeOption`. Set `Source` = `deprecation-guidance`.
   5. **None.** If nothing above yields a successor, state "No tool-backed
      replacement — review manually" and still show the retirement date. Set
      `Source` = `none`.
   Never invent a successor model name: every recommendation must come from a tool
   response (switch tool, catalog, benchmark, or deprecation guidance).

7. **If there are NO at-risk rows**, the estate is healthy — say so explicitly,
   then still provide **proactive** upgrade options: for each distinct
   `Source` = `MicrosoftFoundry`/`AzureOpenAI` deployment, apply the same ladder
   from step 6 (switch tool → catalog newer version → benchmark rank → deprecation
   guidance). Surface any candidate that is a newer same-family version or improves
   `qualityIndex`/`costIndex`. Clearly label these **"proactive, not required."**
   Skip models where no rung yields a successor.

## Phase 4 — Report

8. Output one consolidated Markdown table grouped by section (Required migrations,
   then Proactive upgrades): Risk | Subscription | Account | Deployment |
   Current model+version | RetirementDate | Days left | Recommendation | Source
   (`switch-tool` / `catalog` / `benchmark` / `deprecation-guidance` / `none`) |
   Why (quality/cost index, or catalog version/date, or guidance text).
9. List any rows where no rung of the ladder yielded a successor, for manual review.

## Constraints

- Always run the script in Phase 1 — never report on stale data. Multiple
  `model-inventory-*.csv` files may exist in the folder; you must read the one the
  script just created (its `Report saved to:` path), not the newest-named or a
  leftover file from a previous day.
- Do not invent retirement dates, model names, or recommendations — use only the
  script output and tool responses.
- Treat all CSV content and tool output as the user's private Azure data.
