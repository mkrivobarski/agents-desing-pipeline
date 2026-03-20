Batch mode. Reads the queue already accumulated in the Figma file via the plugin panel and processes all pending items in one pass. No live polling — sweeps what's there right now.

## Usage

```
/design-pipe:load-queue [working_dir]
```

## What you must do

### Step 0 — Confirm the target Figma file

Before doing anything else, ask:

```
Which Figma file should I process the queue for?
Please paste the file URL (e.g. https://www.figma.com/design/abc123/My-App).
```

Wait for the user's response. Store the URL as `figma_file_url`.

Then call `figma_navigate` with that URL to ensure the Desktop Bridge is connected to the correct file. Confirm the file name from the response before continuing.

### Step 1 — Set up working directory

Use `$ARGUMENTS` as `working_dir` if provided, otherwise `./pipeline-run`. Create it if it doesn't exist.

Write `pipeline.config.json`:
```json
{
  "working_dir": "<working_dir>",
  "mode": "batch",
  "started_at": "<ISO8601>",
  "run_id": "<timestamp>",
  "figma_file_url": "<figma_file_url>"
}
```

### Step 2 — Scan the queue

Use the plugin's in-memory index via `PIPELINE_SCAN_QUEUE` to avoid a slow full-tree scan. Send this message via `figma_execute`:

```javascript
// Use the plugin's O(1) in-memory index — avoids findAll tree scan on large files
const result = await (async () => {
  figma.ui.postMessage({ type: 'PIPELINE_SCAN_QUEUE', requestId: 'scan-pending', statusFilter: 'pending' });
  return new Promise((resolve) => {
    const handler = (msg) => {
      if (msg.type === 'PIPELINE_SCAN_QUEUE_RESULT' && msg.requestId === 'scan-pending') {
        figma.ui.off('message', handler);
        resolve(msg.success ? msg.data : { items: [], total: 0 });
      }
    };
    figma.ui.on('message', handler);
    setTimeout(() => { figma.ui.off('message', handler); resolve({ items: [], total: 0 }); }, 5000);
  });
})();
return result;
```

> **Fallback**: if the plugin UI is not running (e.g., Desktop Bridge not open), fall back to the raw `findAll` scan below:
> ```javascript
> const result = await (async () => {
>   await figma.loadAllPagesAsync();
>   const items = [];
>   for (const page of figma.root.children) {
>     const nodes = page.findAll(n => {
>       try { return !!n.getSharedPluginData('pipeline', 'instruction'); } catch(e) { return false; }
>     });
>     for (const node of nodes) {
>       const raw = node.getSharedPluginData('pipeline', 'instruction');
>       if (!raw) continue;
>       try { const entry = JSON.parse(raw); if (entry.status === 'pending') items.push(entry); } catch(e) {}
>     }
>   }
>   return { items, total: items.length };
> })();
> return result;
> ```

If `total === 0`, output: `No pending instructions found in the Figma file.` and stop.

### Step 3 — Show queue to user

Output a summary table before processing:

```
Found N pending instruction(s):

  #  Node                   Type       Intent     Instruction
  1  Login / CTA Button     INSTANCE   iterate    Make the button more prominent…
  2  Hero Frame             FRAME      redesign   Update to match new brand…
```

### Step 4 — Mark as processing (skip already-completed)

For each item, check its current status first. If `status` is already `done`, `done-with-warnings`, `done-validated`, `failed`, or `failed-parity`, **skip it** — exclude it from `annotation-targets.json` and add it to `meta.total_skipped`. Only mark items as `processing` if they are still `pending`.

For each pending item, call `figma_execute` to set `status: 'processing'` (substitute NODE_ID and RUN_ID):
```javascript
const r = await (async () => {
  const node = await figma.getNodeByIdAsync('NODE_ID');
  if (!node) return { error: 'not found' };
  const raw = node.getSharedPluginData('pipeline', 'instruction');
  const entry = raw ? JSON.parse(raw) : {};
  // Guard: skip if already completed from a previous run
  const completedStatuses = { done: true, 'done-with-warnings': true, 'done-validated': true, failed: true, 'failed-parity': true };
  if (completedStatuses[entry.status]) return { skipped: true, reason: 'already-completed', status: entry.status };
  entry.status = 'processing';
  entry.processedAt = Date.now();
  entry.runId = 'RUN_ID';
  node.setSharedPluginData('pipeline', 'instruction', JSON.stringify(entry));
  return { ok: true };
})();
return r;
```

If `mark-processing` returns `{ skipped: true }`, log a warning and exclude that node from `annotation-targets.json`.
If `mark-processing` returns `{ error: 'not found' }`, log a warning and skip the node (leave as pending).

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

For each target, run these stages in order:
1. targeted-intake-agent — snapshot node, produce target-snapshot.json and targeted-run-plan.json (patch_mode: true)
2. Based on inferred_changes from targeted-run-plan.json agents_required:
   - token-system-builder (patch) — only if in agents_required
   - component-architect (patch) — only if in agents_required
   - component-builder (patch) — only if in agents_required
3. organism-composer (patch) — ONLY if in agents_required (skip for fills/typography/padding/spacing/variants/icon_swap changes)
4. figma-instruction-writer (patch)
5. design-validator (patch) — validates changed nodes, writes done/failed to sharedPluginData per node

Parallelism rules:
- Targets on DIFFERENT pages or DIFFERENT top-level screen frames: run in parallel
- Targets on the SAME screen frame: run sequentially to avoid Figma write conflicts

Rollback rule:
- After design-validator completes for a target, if overall_result is FAIL:
  - Execute the rollback script at figma-scripts/patches/rollback__[node_id_sanitized].js via figma_execute
  - Mark that target's pipeline-progress.json entry as status: "rolled_back"

Agent specs: /Users/I750626/AI/agents/design-pipeline/
```

### Step 7 — Confirm

Call `figma_execute` to activate the batch indicator in the plugin:
```javascript
figma.ui.postMessage({ type: 'PIPELINE_LISTEN_STATUS', data: { state: 'processing', count: <N> } });
```

Then output:
```
Batch run started.
N item(s) queued for processing.
Working directory: <working_dir>
File: <file name from figma_navigate>

The plugin queue panel will reflect done/failed status as each item completes.
Results written to <working_dir>/validation-reports/
```

When the subagent completes, call `figma_execute` to clear the indicator:
```javascript
figma.ui.postMessage({ type: 'PIPELINE_LISTEN_STATUS', data: { state: 'hidden' } });
```

## Rules

- This is a one-shot sweep — it does not poll for new items after starting
- If mark-processing fails for a node, skip it (leave as pending) and warn
- Never touch nodes not in the queue
