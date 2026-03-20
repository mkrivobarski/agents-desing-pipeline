Live listen mode. Polls the active Figma file every minute for new instructions submitted via the plugin panel. Each pending item is processed through the patch-mode pipeline as it arrives.

## Usage

```
/design-pipe:listen [working_dir]
```

Stop with `/design-pipe:stop`.

## What you must do

### Step 0 — Confirm the target Figma file

Before doing anything else, ask:

```
Which Figma file should I listen on?
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
  "mode": "listen",
  "started_at": "<ISO8601>",
  "run_id": "<timestamp>",
  "figma_file_url": "<figma_file_url>"
}
```

### Step 2 — Create the polling cron job

Use CronCreate:
- `cron`: `"* * * * *"`
- `recurring`: `true`
- `prompt`: the poll prompt below with `<working_dir>` substituted

**Poll prompt:**
```
Design pipeline listen tick. Working directory: <working_dir>. Figma file: <figma_file_url>. Do the following silently.

1. Call figma_navigate with <figma_file_url> to ensure the Desktop Bridge is scoped to the correct file.

2. Call figma_execute to scan for pending items using the plugin's in-memory index:
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

Fallback if plugin UI not running — use raw findAll scan:
const result = await (async () => {
  await figma.loadAllPagesAsync();
  const items = [];
  for (const page of figma.root.children) {
    const nodes = page.findAll(n => {
      try { return !!n.getSharedPluginData('pipeline', 'instruction'); } catch(e) { return false; }
    });
    for (const node of nodes) {
      const raw = node.getSharedPluginData('pipeline', 'instruction');
      if (!raw) continue;
      try { const e = JSON.parse(raw); if (e.status === 'pending') items.push(e); } catch(e) {}
    }
  }
  return { items, total: items.length };
})();
return result;

3. If total === 0:
   - Call figma_execute: `figma.ui.postMessage({ type: 'PIPELINE_LISTEN_STATUS', data: { state: 'idle' } });`
   - Stop — nothing to do.

4. Items found — update indicator to processing state:
   Call figma_execute: `figma.ui.postMessage({ type: 'PIPELINE_LISTEN_STATUS', data: { state: 'processing', count: <total> } });`

5. For each pending item:
   a. Mark processing via figma_execute (substitute NODE_ID):
   const r = await (async () => {
     const node = await figma.getNodeByIdAsync('NODE_ID');
     if (!node) return { error: 'not found' };
     const raw = node.getSharedPluginData('pipeline', 'instruction');
     const entry = raw ? JSON.parse(raw) : {};
     const completedStatuses = { done: true, 'done-with-warnings': true, 'done-validated': true, failed: true, 'failed-parity': true };
     if (completedStatuses[entry.status]) return { skipped: true, status: entry.status };
     entry.status = 'processing';
     entry.processedAt = Date.now();
     node.setSharedPluginData('pipeline', 'instruction', JSON.stringify(entry));
     if (typeof _pipelineIndex !== 'undefined') _pipelineIndex['NODE_ID'] = entry;
     return { ok: true };
   })();
   return r;

   b. If { skipped: true }, skip this item — it was already processed. Continue to next.

   c. Write a single-entry annotation-targets.json to <working_dir>. Infer scope: FRAME/SECTION→frame, COMPONENT/COMPONENT_SET→component, INSTANCE→instance, else→element.

   d. Launch Agent (subagent_type: general-purpose) with prompt:
      "Run the patch-mode pipeline for one target. Working directory: <working_dir>. Figma file: <figma_file_url>. annotation-targets.json is already written.
      Run stages in order: targeted-intake-agent → agents from targeted-run-plan.json agents_required (token-system-builder/component-architect/component-builder as needed) → organism-composer ONLY if in agents_required → figma-instruction-writer → design-validator.
      design-validator writes done/failed back to sharedPluginData and updates _pipelineIndex directly.
      Rollback: if design-validator returns overall_result FAIL, execute figma-scripts/patches/rollback__[node_id_sanitized].js via figma_execute.
      Agent specs: /Users/I750626/AI/agents/design-pipeline/"

6. After dispatching all agents:
   Call figma_execute: `figma.ui.postMessage({ type: 'PIPELINE_LISTEN_STATUS', data: { state: 'listening' } });`

7. Log: [design-pipe:listen] Dispatched N item(s) at <time>
```

### Step 3 — Write job ID

Write the CronCreate job ID to `<working_dir>/.listen_job_id`.

### Step 4 — Confirm

Call `figma_execute` to activate the listen indicator in the plugin:
```javascript
figma.ui.postMessage({ type: 'PIPELINE_LISTEN_STATUS', data: { state: 'listening' } });
```

Then output:
```
Listening on: <file name from figma_navigate>
Working directory: <working_dir>
Polling every minute — submit from the Figma plugin panel to queue work.
Stop with /design-pipe:stop
```

## Rules

- Dispatch multiple pending items as parallel agents per tick
- Never mark items done — that is design-validator's responsibility
- If figma_execute fails (plugin not open), log a warning but keep the cron running
- Cron auto-expires after 3 days per Claude Code session limits
