# Table Migration Review (PowerShell)

PowerShell equivalent of the Table Migration Manager web app. Discovers classic V1
custom log tables in a Microsoft Sentinel workspace, assesses dependency impact
across all content types, and maps tables to Content Hub solutions with CCF /
Azure Functions connector classification.

## Requirements

- PowerShell **7.0 or later** (uses null-coalescing operator and `ForEach-Object -Parallel`)
- `Az.Accounts` **2.13.0 or later**
- Read access to the target Sentinel workspace (Reader role is sufficient for
  the discovery and impact scan; Security Reader for Content Hub queries)

## Usage

### Interactive

```powershell
./Invoke-TableMigrationReview.ps1
```

Prompts for subscription ID, resource group, and workspace name. Writes all
reports to `./migration-report/`.

### Scripted

```powershell
./Invoke-TableMigrationReview.ps1 `
    -SubscriptionId '00000000-0000-0000-0000-000000000000' `
    -ResourceGroupName 'rg-sentinel' `
    -WorkspaceName 'ws-sentinel' `
    -OutputPath './reports/2026-04' `
    -NonInteractive
```

### Pipeline

The script writes its summary object to the pipeline — capture it for further processing:

```powershell
$result = ./Invoke-TableMigrationReview.ps1 -SubscriptionId $sub -ResourceGroupName $rg -WorkspaceName $ws

$result.ClassicTables | Where-Object { $_.RetentionInDays -gt 90 }
$result.Impacts       | Where-Object { $_.TotalImpacted -gt 0 } | Sort-Object TotalImpacted -Descending
$result.SolutionMatches | Where-Object { $_.MatchCount -eq 0 }
```

## Output

Each run writes four artefacts to `OutputPath`:

| File                    | Purpose                                                   |
| ----------------------- | --------------------------------------------------------- |
| `tables.csv`            | Discovered classic tables with schema metadata            |
| `impact.csv`            | Flat list of every impacted content item, one row each    |
| `solution-matches.csv`  | Table → Content Hub solution matches with install status  |
| `report.json`           | Everything combined, suitable for further tooling         |
| `report.html`           | Self-contained interactive HTML report                    |

Plus the full `PSCustomObject` is returned to the pipeline.

## The three steps

1. **Discover** — Queries the workspace `tables` API for `CustomLog` / `Classic`
   tables and returns schema metadata.

2. **Assess impact** — For each classic table, scans every Analytics Rule,
   Hunting Query, Parser, Saved Search, Workbook (full serialised JSON walk),
   SOAR Playbook (Logic App definition), and DCR transform KQL for references.

3. **Map to Content Hub** — Uses the static solution mapping
   (`../src/lib/data/solution-mapping.json`, 826 tables → 495 solutions,
   sourced from the [Azure-Sentinel Solutions Analyzer](https://github.com/Azure/Azure-Sentinel/tree/master/Tools/Solutions%20Analyzer)).
   Each matched solution is classified by connector kind:

   | Kind             | Indicator                                                    | Recommendation |
   | ---------------- | ------------------------------------------------------------ | -------------- |
   | `CCF`            | Connector ID ends in `CCP` / `CCF` / `Definition`           | Modern — preferred |
   | `AzureFunctions` | Name contains `Serverless` / `AzureFunction` / `Polling`    | Legacy — migrate to CCF |
   | `AMA`            | Name ends in `Ama`                                           | Modern — preferred |
   | `Platform`       | Microsoft-native (`Azure*`, `Office*`, `Defender*`, …)      | Platform-managed |
   | `Agent`          | `CEF` / `Syslog`                                             | Agent-based |
   | `Legacy`         | Anything else                                                | Review manually |

   If no solution matches a table at all, the report flags it with a
   recommendation to raise a Feature Request with your CSAM / SSP.

## Data source

The Content Hub solution mapping is shared with the Next.js web app
(`src/lib/data/solution-mapping.json`) and refreshed weekly by the
`update-solution-mapping.yml` GitHub Action. Both tools stay in sync
automatically.

## Copilot Agent

This repository includes a **GitHub Copilot custom agent** that guides you through
deploying and running the script. In VS Code with Copilot enabled:

1. Open this repository.
2. Open Copilot Chat and select the **sentinel-migration-assistant** agent from the agent dropdown.
3. Ask it to help you run the migration review — it will check prerequisites, walk you through parameters, and help interpret results.

The agent profile lives at `.github/agents/sentinel-migration-assistant.md`.

## Troubleshooting

**`InsufficientPermissions` on Content Hub query**
Requires `Microsoft.SecurityInsights/contentPackages/read` — included in
Microsoft Sentinel Reader role.

**`401 Unauthorized` from ARM**
Run `Connect-AzAccount` and ensure the context points at the tenant containing
the workspace. Token is refreshed automatically on each REST call.

**Workbooks API returns nothing**
The subscription-level query requires `Microsoft.Insights/workbooks/read`. If
you're scoped to RG-only, workbooks may be missed.
