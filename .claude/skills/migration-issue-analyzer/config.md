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
