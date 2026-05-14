---
name: migration-issue-analyzer
description: >
  Analyze migration run issues from Kusto, group them semantically, and produce
  actionable improvement instructions. Use this skill whenever the user wants to
  analyze a migration run, investigate migration issues, group or categorize
  migration errors, research migration failures, or generate migration
  improvement guidance. Triggers on phrases like "analyze run", "check migration
  issues", "what went wrong in run X", "group the issues", or any mention of a
  runId combined with migration analysis intent.
---

# Migration Issue Analyzer

You are an expert .NET-to-Azure migration analyst. Given a migration **runId**,
you will retrieve all issues from Kusto, semantically group them, and produce
deep-research-backed instructions to improve migration quality.

## Prerequisites

- **azure-kusto** skill must be available for querying migration data
- **Web search** capability for researching issues and finding up-to-date guidance

## Workflow

### Step 1: Setup workspace

1. Ask the user for the **runId** if not already provided.
2. Create a timestamped output folder:
   ```
   <runId>-<YYYYMMDDHHMI>
   ```
   For example: `abc123-202605141545`

   Use the current date and time (24-hour format, no separators except the
   dash between runId and timestamp). Create this folder in the current working
   directory.

### Step 2: Query Kusto for issues

Run the following Kusto query using skill `azure-kusto` against 

- cluster: https://springbootmigration.eastus.kusto.windows.net
- database: benchmark
- Query

```kql
summarize_benchmark_run(<runId>)
| mv-expand Issues to typeof(string)
| where isnotempty(Issues)
| project Issues
```

Save the raw query results to `<folder>/issues.txt` — one issue per line, with
all returned columns preserved. If the query returns structured data (e.g., a
table), save it in a readable tab-separated or pipe-separated format with a
header row.

If the query returns zero results, tell the user and stop.

### Step 3: Semantic grouping

Read `<folder>/issues.txt` and analyze every issue. Group them **semantically**
— not by exact string match, but by the underlying problem they represent. Two
issues with different wording but the same root cause belong in one group.

Guidelines for grouping:
- Look at the error message meaning, not just keywords
- Consider the component/module context when deciding if issues are related
- A group should represent a single actionable problem area
- If an issue doesn't fit any group, it can be its own group (count of 1)

Write `<folder>/grouped-issues.txt` in this format, sorted descending by count:

```
Count | Issue Group
------|-----------
  42  | <concise description of the grouped issue>
  31  | <concise description of the grouped issue>
  ...
```

Include ALL groups, not just the top ones. After the table, add a brief summary
line: total number of issues, total number of groups, and what percentage the
top 10 groups cover.

### Step 4: Deep research and instruction generation

Take the **top 10** groups from `grouped-issues.txt`. For each one:

1. **Understand the issue** — Read the individual issues in that group from
   `issues.txt` to understand the full picture: what component fails, what the
   error messages say, what patterns exist.

2. **Research** — Search the web for:
   - The specific error messages or failure patterns
   - Microsoft's official .NET-to-Azure migration documentation
   - Azure service-specific migration guides relevant to the issue
   - Known issues, breaking changes, or compatibility notes
   - Community solutions (Stack Overflow, GitHub issues, blog posts)
   - Any recent updates or changes that may affect migration behavior

3. **Analyze root cause** — Based on your research, identify:
   - Why this issue occurs during .NET-to-Azure migration
   - What the underlying cause is (API incompatibility, configuration gap,
     deprecated feature, missing dependency, etc.)
   - Whether this is a known limitation or a fixable problem

4. **Compose improvement instruction** — Write a clear, actionable instruction
   that a migration tool or engineer can follow to prevent or resolve this
   issue. The instruction should be:
   - Specific enough to act on (not "handle errors better")
   - Grounded in the research you did (reference docs/sources)
   - Aware of Azure best practices
   - Practical for automated migration tooling where applicable

Save the output to `<folder>/migration-instructions.md` using this structure:

```markdown
# Migration Improvement Instructions

> Analysis of run: <runId>
> Date: <timestamp>
> Total issues: <N> | Groups: <N> | Top 10 coverage: <X%>

---

## 1. <Issue Group Title> (Count: N)

### Problem
<What goes wrong and how it manifests>

### Root Cause
<Why this happens during .NET-to-Azure migration>

### Research Findings
<Key findings from web research, with source references>

### Improvement Instruction
<Clear, actionable instruction to prevent/resolve this issue>

---

## 2. <Issue Group Title> (Count: N)
...
```

### Step 5: Summary

After generating the instructions file, present the user with:
- Path to all three output files
- A brief summary of the top 3 most impactful issue groups
- Any issues that could not be fully researched (and why)
- Suggested next steps (e.g., "review instructions", "update migration rules",
  "file bugs for unresolved issues")

## Tips

- If the skill `azure-kusto` is not available, ask the user how to connect or
  whether they can provide the issues data directly (e.g., paste or file).
- If web search is unavailable, still produce instructions based on your
  existing knowledge, but flag that research was limited.
- For very large result sets (>1000 issues), mention the scale to the user and
  confirm they want to proceed, as grouping will take time.
