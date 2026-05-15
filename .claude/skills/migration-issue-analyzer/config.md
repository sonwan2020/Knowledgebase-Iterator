# Migration Issue Analyzer — Configuration

## Kusto Connection

- **Cluster:** `https://springbootmigration.eastus.kusto.windows.net`
- **Database:** `benchmark`

## Query Template

```kql
summarize_benchmark_run(<runId>)
| mv-expand Issues to typeof(string)
| where isnotempty(Issues)
| project Issues
```

Replace `<runId>` with the actual run identifier provided by the user.

## Knowledgebase Source

- **Repository:** `devdiv-azure-service-dmitryr/azure-java-migration-copilot-vscode-extension`
- **Branch:** `sonwan/dotnet-kb-skill-based`
- **Path:** `kb/dotnet/dotnet-azure-storage-blob.md`
- **GitHub Account:** `sonwan_microsoft` (required for private repo access)

Fetch command:
```bash
gh auth switch --user <GitHub Account>
gh api repos/<Repository>/contents/<Path>?ref=<Branch> --jq '.content' | base64 -d
```
