Start a new design project from scratch. Accepts either a written brief or an existing Figma file as the starting point, then runs the full upstream pipeline to produce a complete design system and screens.

## Usage

```
/design-pipe:new [working_dir]
```

You will be asked for a brief or Figma file URL if not provided in `$ARGUMENTS`.

## What you must do

### Step 1 — Gather context

If `$ARGUMENTS` contains a Figma URL or a path to a brief file, use it. Otherwise ask the user:

1. **Starting point** — written brief (paste it) or existing Figma file URL?
2. **Figma file URL** — where to build the designs (required even if starting from brief)
3. **Scope** — full product, single flow, or single screen?

### Step 1b — Check for interrupted run (session recovery)

Before writing `pipeline.config.json`, check whether `<working_dir>/pipeline-progress.json` already exists.

If it exists and `stages.design-validator` is NOT `"done"`:
- Output:
  ```
  Found an interrupted pipeline run in <working_dir>.
  Last completed stage: <last stage with status "done">
  Resuming from: <first stage not yet "done">
  ```
- Launch an Agent (subagent_type: `general-purpose`) with prompt:
  ```
  Session recovery: resume interrupted full pipeline run.
  Working directory: <working_dir>
  Read pipeline-progress.json to determine which stages completed and which was interrupted.
  Resume from the first stage whose status is not "done". Skip all stages already marked "done".
  Each agent reads pipeline-progress.json on startup and skips if its stage is already complete.
  Agent specs: /Users/I750626/AI/agents/design-pipeline/
  ```
- Stop here — do not re-run gather context or overwrite `pipeline.config.json`.

If `pipeline-progress.json` does not exist, or `stages.design-validator === "done"`, continue to Step 2.

### Step 2 — Set up working directory

Use `$ARGUMENTS` as `working_dir` if it is a path (not a URL), otherwise `./pipeline-run`. Create it if needed.

Write `pipeline.config.json`:
```json
{
  "working_dir": "<working_dir>",
  "figma_url": "<url>",
  "mode": "new",
  "scope": "<scope>",
  "started_at": "<ISO8601>",
  "run_id": "<timestamp>"
}
```

If starting from a brief, write it to `<working_dir>/brief.md`.

### Step 3 — Launch the full pipeline

Launch an Agent (subagent_type: `general-purpose`):

```
Run a full new-project pipeline.
Working directory: <working_dir>
Figma URL: <figma_url>
Brief: <brief or "see brief.md">
Scope: <scope>

Run the following agents in order. Each reads outputs from the previous.
Each agent must check pipeline-progress.json on startup — if its stage is already marked "done", skip it and proceed to the next.
Agent specs: /Users/I750626/AI/agents/design-pipeline/

1. requirements-analyst
2. flow-architect
3. journey-mapper
4. wireframe-strategist
5. lo-fi-builder
6. token-system-builder
7. component-architect
8. component-builder
9. organism-composer
10. figma-instruction-writer
11. design-validator

Write all artifacts to <working_dir>.
```

### Step 4 — Confirm

```
New project pipeline started.
Working directory: <working_dir>
Figma: <url>

Running all pipeline stages end-to-end.
Use /design-pipe:listen to process plugin instructions during the run.
Use /design-pipe:review once complete to audit the output.
```

## Rules

- A Figma URL is required — do not proceed without one
- If brief is under 20 words, warn the user but proceed
- Write all artifacts to `working_dir`
- Session recovery (Step 1b) takes priority — always check for interrupted runs before starting fresh


## Usage

```
/design-pipe:new [working_dir]
```

You will be asked for a brief or Figma file URL if not provided in `$ARGUMENTS`.

## What you must do

### Step 1 — Gather context

If `$ARGUMENTS` contains a Figma URL or a path to a brief file, use it. Otherwise ask the user:

1. **Starting point** — written brief (paste it) or existing Figma file URL?
2. **Figma file URL** — where to build the designs (required even if starting from brief)
3. **Scope** — full product, single flow, or single screen?

### Step 2 — Set up working directory

Use `$ARGUMENTS` as `working_dir` if it is a path (not a URL), otherwise `./pipeline-run`. Create it if needed.

Write `pipeline.config.json`:
```json
{
  "working_dir": "<working_dir>",
  "figma_url": "<url>",
  "mode": "new",
  "scope": "<scope>",
  "started_at": "<ISO8601>",
  "run_id": "<timestamp>"
}
```

If starting from a brief, write it to `<working_dir>/brief.md`.

### Step 3 — Launch the full pipeline

Launch an Agent (subagent_type: `general-purpose`):

```
Run a full new-project pipeline.
Working directory: <working_dir>
Figma URL: <figma_url>
Brief: <brief or "see brief.md">
Scope: <scope>

Run the following agents in order. Each reads outputs from the previous.
Agent specs: /Users/I750626/AI/agents/design-pipeline/

1. requirements-analyst
2. flow-architect
3. journey-mapper
4. wireframe-strategist
5. lo-fi-builder
6. token-system-builder
7. component-architect
8. component-builder
9. organism-composer
10. figma-instruction-writer
11. design-validator

Write all artifacts to <working_dir>.
```

### Step 4 — Confirm

```
New project pipeline started.
Working directory: <working_dir>
Figma: <url>

Running all pipeline stages end-to-end.
Use /design-pipe:listen to process plugin instructions during the run.
Use /design-pipe:review once complete to audit the output.
```

## Rules

- A Figma URL is required — do not proceed without one
- If brief is under 20 words, warn the user but proceed
- Write all artifacts to `working_dir`
