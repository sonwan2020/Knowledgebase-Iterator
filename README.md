# Knowledgebase-Iterator

Automated analysis of .NET-to-Azure migration runs — queries issues from Kusto, groups them semantically, and generates research-backed improvement instructions to strengthen migration knowledge base (KB) rules.

## What It Does

Given one or more migration **run IDs**, this tool:

1. **Queries Azure Data Explorer (Kusto)** for all issues produced during migration benchmark runs (supports multiple run IDs — merges and deduplicates across runs)
2. **Semantically groups** issues by root cause (not string matching), tracking severity and KB rule references
3. **Researches each group** against Microsoft docs, SDK references, and community solutions
4. **Generates actionable instructions** to create or update KB rules, improving future migration quality

## Output Structure

Each analysis produces a timestamped folder containing:
- **Single run**: `<runId>-<YYYYMMDDHHMI>/`
- **Multiple runs**: `multi-<YYYYMMDDHHMI>/`

| File | Description |
|------|-------------|
| `issues.txt` | Raw issues from Kusto (merged and deduplicated when using multiple runs) |
| `grouped-issues.txt` | Semantic groups sorted by severity and count |
| `migration-instructions.md` | Deep-research instructions per group with KB rule recommendations |

## Prerequisites

- [GitHub Copilot CLI](https://docs.github.com/en/copilot) with agent mode
- Azure MCP Server (configured in `.mcp.json`) for Kusto queries
- Access to the migration benchmark Kusto cluster
- The `azure-kusto` skill (user-level) for query execution

## Usage

In Copilot CLI agent mode, invoke the skill by asking:

```
analyze run <runId>
analyze runs <runId1> <runId2> <runId3>
```

Or any variation like:
- "check migration issues for run 25838804895"
- "what went wrong in run 25838769390"
- "group the issues from run <id>"
- "analyze these runs: 123, 456, 789"

## Configuration

Edit `.claude/skills/migration-issue-analyzer/config.md` to change:

- **Cluster** — Kusto cluster URL
- **Database** — database name
- **Query template** — KQL query with `<runId>` placeholder

## Example Results

From a typical run analysis:

```
Count | Severity | KB Rule | Issue Group
------|----------|---------|------------
  11  | Major    | KB-10   | Upload-semantics drift (missing overwrite:true)
   5  | Major    | KB-11   | BlobDownloadStreamingResult not disposed
   1  | Critical | —       | Non-existent SDK type causes compile error
```

## Architecture

```
.claude/skills/migration-issue-analyzer/
├── SKILL.md       # Skill definition and workflow
└── config.md      # Kusto connection and query config

.mcp.json          # Azure MCP Server configuration (Kusto namespace)
```

## License

Internal tool — not for public redistribution.
