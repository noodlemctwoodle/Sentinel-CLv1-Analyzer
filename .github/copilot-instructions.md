# Sentinel CLv1 Analyzer

PowerShell tool that discovers classic V1 custom log tables in a Microsoft Sentinel workspace, assesses dependency impact, and maps to Content Hub solutions.

## Repository structure

```
Invoke-TableMigrationReview.ps1   # Main script — entry point
Templates/report.html.template    # Self-contained HTML report template
data/solution-mapping.json        # Static table-to-solution lookup (auto-updated weekly)
data/update-solution-mapping.mjs  # Node.js script to refresh mapping from upstream CSVs
.github/workflows/update-solution-mapping.yml  # Weekly GH Action to keep mapping current
.github/agents/sentinel-migration-assistant.md # Copilot custom agent for deployment guidance
```

## Prerequisites

- PowerShell 7.0+
- Az.Accounts module 2.13.0+
- Reader role on the target Sentinel workspace (Security Reader for Content Hub)

## Quick start

```powershell
# Interactive
./Invoke-TableMigrationReview.ps1

# Scripted
./Invoke-TableMigrationReview.ps1 `
    -SubscriptionId '<sub-id>' `
    -ResourceGroupName '<rg>' `
    -WorkspaceName '<ws>' `
    -OutputPath './reports' `
    -NonInteractive
```

## Build and validation

No build step required. Run the script directly with PowerShell 7+. Validate with:

```powershell
pwsh -NoProfile -Command "& { Get-Help ./Invoke-TableMigrationReview.ps1 -Full }"
```

## Key conventions

- PowerShell uses `#Requires` statements for version and module dependencies
- All Azure API calls go through `Invoke-ArmRequest` using ARM bearer tokens
- Solution mapping data comes from the Azure-Sentinel Solutions Analyzer upstream
- Output defaults to `./migration-report/` — this directory is git-ignored
