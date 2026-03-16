Batch mode. Reads the queue already accumulated in the Figma file via the plugin panel and processes all pending items in one pass. No live polling — sweeps what's there right now.

## Usage

```
/design-pipe:load-queue [working_dir]
```

## What you must do

### Step 1 — Set up working directory

Use `$ARGUMENTS` as `working_dir` if provided, otherwise `./pipeline-run`. Create it if it doesn't exist.

Write `pipeline.config.json`:
```json
{
  "working_dir": "<working_dir>",
  "mode": "batch",
  "started_at": "<ISO8601>",
  "run_id": "<timestamp>"
}
```

### Step 2 — Scan the queue

Call `figma_execute`:
```javascript
(async () => {
  await figma.loadAllPagesAsync();
  const items = [];
  for (const page of figma.root.children) {
    const nodes = page.findAll(n => {
      try { return !!n.getSharedPluginData('pipeline', 'instruction'); } catch(e) { return false; }
    });
    for (const node of nodes) {
      const raw = node.getSharedPluginData('pipeline', 'instruction');
      if (!raw) continue;
      try {
        const entry = JSON.parse(raw);
        if (entry.status === 'pending') items.push(entry);
      } catch(e) {}
    }
  }
  return { items, total: items.length };
})()
```

If `total === 0`, output: `No pending instructions found in the Figma file.` and stop.

### Step 3 — Show queue to user

Output a summary table before processing:

```
Found N pending instruction(s):

  #  Node                   Type       Intent     Instruction
  1  Login / CTA Button     INSTANCE   iterate    Make the button more prominent…
  2  Hero Frame             FRAME      redesign   Update to match new brand…
```

### Step 4 — Mark all as processing

For each item, call `figma_execute` to set `status: 'processing'` (substitute NODE_ID and RUN_ID):
```javascript
(async () => {
  const node = await figma.getNodeByIdAsync('NODE_ID');
  if (!node) return { error: 'not found' };
  const raw = node.getSharedPluginData('pipeline', 'instruction');
  const entry = raw ? JSON.parse(raw) : {};
  entry.status = 'processing';
  entry.processedAt = Date.now();
  entry.runId = 'RUN_ID';
  node.setSharedPluginData('pipeline', 'instruction', JSON.stringify(entry));
  return { ok: true };
})()
```

### Step 5 — Write annotation-targets.json

Write all targets to `<working_dir>/annotation-targets.json`. Infer scope from nodeType: FRAME/SECTION→frame, COMPONENT/COMPONENT_SET→component, INSTANCE→instance, else→element.

```json
{
  "meta": {
    "scanned_at": "<ISO8601>",
    "total_pending": N,
    "total_skipped": 0,
    "mode": "batch",
    "run_id": "<run_id>"
  },
  "targets": [...]
}
```

### Step 6 — Run the pipeline

Launch an Agent (subagent_type: `general-purpose`):

```
Run a batch patch-mode pipeline sweep.
Working directory: <working_dir>
annotation-targets.json is written with N targets.

For each target, run in order:
1. targeted-intake-agent — snapshot node, produce target-snapshot.json and targeted-run-plan.json (patch_mode: true)
2. Based on inferred_changes: token-system-builder (patch), component-architect (patch), component-builder (patch) as needed
3. organism-composer (patch)
4. figma-instruction-writer (patch)
5. design-validator (patch) — writes done/failed to sharedPluginData per node

Process targets in parallel where safe to do so.
Agent specs: /Users/I750626/AI/agents/design-pipeline/
```

### Step 7 — Confirm

```
Batch run started.
N item(s) queued for processing.
Working directory: <working_dir>

The plugin queue panel will reflect done/failed status as each item completes.
Results written to <working_dir>/validation-reports/
```

## Rules

- This is a one-shot sweep — it does not poll for new items after starting
- If mark-processing fails for a node, skip it (leave as pending) and warn
- Never touch nodes not in the queue
