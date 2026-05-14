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
you will retrieve all issues from Kusto, semantically group them (preserving
severity and KB rule references), research every group online, and produce
improvement instructions that can be used to **update and strengthen the
migration knowledge base (KB rules)**. The existing KB is the improvement
target — always search the web for the latest and best solutions regardless
of whether an issue cites a KB rule.

## Prerequisites

- **azure-kusto** skill must be available for querying migration data
- **Web search** capability for researching **every** issue group (always
  search online — the KB is the improvement target, not a trusted source)

## Configuration

Settings are stored in `config.md` in this skill's folder
(`.claude/skills/migration-issue-analyzer/config.md`). Read it at the start of
every run to get:

- **Cluster** — the Kusto cluster URL
- **Database** — the database name
- **Query template** — the KQL query to execute (with `<runId>` placeholder)

If `config.md` is missing or incomplete, ask the user for the missing values
before proceeding.

## Workflow

### Step 1: Setup workspace

1. Ask the user for the **runId** if not already provided.
2. Read `config.md` to load cluster, database, and query template.
3. Create a timestamped output folder in the current working directory:
   ```
   <runId>-<YYYYMMDDHHMI>
   ```
   For example: `abc123-202605141545`

   Use the current date and time (24-hour format, no separators except the
   dash between runId and timestamp).

### Step 2: Query Kusto for issues

Run the KQL query from `config.md` using skill `azure-kusto`, substituting the
user's runId into the `<runId>` placeholder.

Save the raw query results to `<folder>/issues.txt` — one issue per line, with
all returned columns preserved. If the query returns structured data (e.g., a
table), save it in a readable tab-separated or pipe-separated format with a
header row.

If the query returns zero results, tell the user and stop.

### Step 3: Semantic grouping with severity and KB rule tracking

Read `<folder>/issues.txt` and analyze every issue. Group them **semantically**
— not by exact string match, but by the underlying problem they represent. Two
issues with different wording but the same root cause belong in one group.

While grouping, extract and preserve two additional dimensions per issue:

- **Severity** — look for severity indicators in the issue text (Critical,
  Major, Minor, Warning). If not explicitly stated, default to Major.
- **KB Rule** — look for references like "KB rule 10", "KB Rule 7",
  "knowledge base rule N", etc. Extract the rule number. If no KB rule is
  referenced, mark as "—".

Guidelines for grouping:
- Look at the error message meaning, not just keywords
- Consider the component/module context when deciding if issues are related
- A group should represent a single actionable problem area
- If an issue doesn't fit any group, it can be its own group (count of 1)

Write `<folder>/grouped-issues.txt` in this format.

**Sort order:** Critical severity first, then Major, then Minor/Warning. Within
each severity level, sort descending by count.

```
Count | Severity | KB Rule | Issue Group
------|----------|---------|------------
   1  | Critical | —       | <concise description>
  11  | Major    | KB-10   | <concise description>
   5  | Major    | KB-11   | <concise description>
   1  | Major    | KB-7    | <concise description>
   1  | Major    | —       | <concise description>
```

- **Severity** = highest severity found among issues in that group
- **KB Rule** = the rule number cited by issues in the group (e.g., `KB-10`),
  or `—` if no rule is cited

Include ALL groups, not just the top ones. After the table, add a summary:
- Total number of issues and total number of groups
- Percentage of issues covered by the top 3 groups
- **KB coverage**: how many groups reference a known KB rule vs. how many are
  novel (no KB rule)

### Step 4: Deep research and instruction generation

Process **all** groups from `grouped-issues.txt`. For each group, **always
perform web research** regardless of whether the group cites an existing KB
rule. The goal is to find the best, most up-to-date solutions so the KB rules
can be improved or new rules can be created.

For each group:

1. **Understand the issue** — Read the individual issues in that group from
   `issues.txt` to understand the full picture: what component fails, what the
   error messages say, what patterns exist.

2. **Research online** — Search the web for:
   - The specific error messages or failure patterns
   - Microsoft's official .NET-to-Azure migration documentation
   - Azure service-specific migration guides relevant to the issue
   - Known issues, breaking changes, or compatibility notes
   - Community solutions (Stack Overflow, GitHub issues, blog posts)
   - Any recent updates or changes that may affect migration behavior
   - **Latest SDK versions and API changes** that may supersede current KB guidance

3. **Analyze root cause** — Based on your research, identify:
   - Why this issue occurs during .NET-to-Azure migration
   - What the underlying cause is (API incompatibility, configuration gap,
     deprecated feature, missing dependency, etc.)
   - Whether this is a known limitation or a fixable problem

4. **Compare with existing KB** — If the group cites a KB rule, compare what
   your research found against what the current KB rule recommends:
   - Is the KB rule's guidance still accurate and complete?
   - Are there better approaches available now (newer APIs, updated patterns)?
   - Does the KB rule miss edge cases that the research uncovered?
   - Note any gaps or improvements needed in the Research Findings section.

5. **Compose improvement instruction** — Write a clear, actionable instruction
   that can be used to **create or update a KB rule**. The instruction should:
   - Be specific enough to act on (not "handle errors better")
   - Be grounded in the research you did (reference docs/sources)
   - Follow Azure best practices
   - Be practical for automated migration tooling
   - If updating an existing KB rule, explicitly state what should change

Save the output to `<folder>/migration-instructions.md` using this structure:

```markdown
# Migration Improvement Instructions

> Analysis of run: <runId>
> Date: <timestamp>
> Total issues: <N> | Groups: <N> | KB-documented: <N> | Novel: <N>

---

## 1. [Critical] <Issue Group Title> (Count: N)

### Problem
<What goes wrong and how it manifests>

### Root Cause
<Why this happens during .NET-to-Azure migration>

### Research Findings
<Key findings from web research with source references>

### KB Rule Status
<One of:>
- "**Existing: KB Rule N** — <assessment of current rule accuracy>. Suggested
  updates: <what should change, if anything>."
- "**New rule needed** — no existing KB rule covers this issue."

### Improvement Instruction
<Clear, actionable instruction to create or update a KB rule for this issue>

---

## 2. [Major] <Issue Group Title> (Count: N)
...
```

The severity badge (`[Critical]`, `[Major]`, etc.) goes in the section header.
Sections are ordered to match the sort from `grouped-issues.txt` (Critical
first, then by count within severity).

### Step 5: Summary

After generating the instructions file, present the user with:
- Path to all three output files
- A brief summary of the top 3 most impactful issue groups
- **KB coverage**: "X of Y groups map to existing KB rules; Z groups need new
  KB rules"
- **KB update recommendations**: which existing rules need updates based on
  research findings, and which new rules should be created
- Any issues that could not be fully researched (and why)
- Suggested next steps (e.g., "update KB rule 10 with the findings",
  "create new KB rules for novel groups", "re-run benchmark after KB updates")

## Tips

- If the skill `azure-kusto` is not available, ask the user how to connect or
  whether they can provide the issues data directly (e.g., paste or file).
- If web search is unavailable, still produce instructions based on your
  existing knowledge, but flag that research was limited.
- For very large result sets (>1000 issues), mention the scale to the user and
  confirm they want to proceed, as grouping will take time.
- To change the Kusto cluster, database, or query, edit `config.md` in this
  skill's folder.
