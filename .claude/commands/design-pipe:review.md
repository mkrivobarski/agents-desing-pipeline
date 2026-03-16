Run a design review pass on a Figma file. Audits screens for accessibility, consistency, token adherence, spacing, typography, and design debt. Produces a ranked findings report without modifying anything in Figma.

## Usage

```
/design-pipe:review <figma_url> [working_dir]
```

- `figma_url` — required. The Figma file to review.
- `working_dir` — where to write the review report (default: `./pipeline-review`)

## What you must do

### Step 1 — Parse arguments

From `$ARGUMENTS`:
- Extract the Figma URL (any argument starting with `https://`)
- Extract `working_dir` (any non-URL argument, default `./pipeline-review`)

If no Figma URL is found, ask the user for one before proceeding.

### Step 2 — Set up working directory

Create `working_dir` if it doesn't exist.

Write `pipeline.config.json`:
```json
{
  "working_dir": "<working_dir>",
  "figma_url": "<url>",
  "mode": "review",
  "started_at": "<ISO8601>",
  "run_id": "<timestamp>"
}
```

### Step 3 — Snapshot the file

Call `figma_get_file_data` (or `figma_execute`) to get a summary of the file structure — page names, top-level frame names, approximate screen count. Report this to the user:

```
Reviewing: <file name>
Pages: N  |  Screens: ~N
```

### Step 4 — Launch the review agent

Launch an Agent (subagent_type: `general-purpose`):

```
Run a full design review on a Figma file.
Working directory: <working_dir>
Figma URL: <figma_url>

Run the design-review-agent. It will:
- Audit all screens for accessibility, consistency, token adherence, spacing, typography, and design debt
- Surface ranked, actionable findings
- Identify patterns that should be abstracted into components
- Write review-report.json and review-summary.md to <working_dir>

Agent spec: /Users/I750626/AI/agents/design-pipeline/design-review-agent.md

Do not modify anything in Figma — this is a read-only audit.
```

### Step 5 — Confirm

```
Design review started.
File: <figma_url>
Working directory: <working_dir>

Review report will be written to <working_dir>/review-summary.md
Nothing will be modified in Figma.
```

## Rules

- Read-only — never call figma_execute scripts that write to nodes during a review run
- If the file is very large (>50 screens), warn the user the review may take a while
- Write all output to `working_dir`
