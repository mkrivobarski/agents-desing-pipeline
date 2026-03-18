Live listen mode. Polls the active Figma file every minute for new instructions submitted via the plugin panel. Each pending item is processed through the patch-mode pipeline as it arrives.

## Usage

```
/design-pipe:listen [working_dir]
```

Stop with `/design-pipe:stop`.

## What you must do

### Step 1 — Set up working directory

Use `$ARGUMENTS` as `working_dir` if provided, otherwise `./pipeline-run`. Create it if it doesn't exist.

Write `pipeline.config.json`:
```json
{
  "working_dir": "<working_dir>",
  "mode": "listen",
  "started_at": "<ISO8601>",
  "run_id": "<timestamp>"
}
```

### Step 2 — Create the polling cron job

Use CronCreate:
- `cron`: `"* * * * *"`
- `recurring`: `true`
- `prompt`: the poll prompt below with `<working_dir>` substituted

**Poll prompt:**
```
Design pipeline listen tick. Working directory: <working_dir>. Do the following silently.

1. Call figma_execute to scan for pending items:
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

2. If total === 0, stop — nothing to do.

3. For each pending item:
   a. Mark processing via figma_execute (substitute NODE_ID):
   const r = await (async () => {
     const node = await figma.getNodeByIdAsync('NODE_ID');
     if (!node) return { error: 'not found' };
     const raw = node.getSharedPluginData('pipeline', 'instruction');
     const entry = raw ? JSON.parse(raw) : {};
     entry.status = 'processing';
     entry.processedAt = Date.now();
     node.setSharedPluginData('pipeline', 'instruction', JSON.stringify(entry));
     return { ok: true };
   })();
   return r;

   b. Write a single-entry annotation-targets.json to <working_dir>. Infer scope: FRAME/SECTION→frame, COMPONENT/COMPONENT_SET→component, INSTANCE→instance, else→element.

   c. Launch Agent (subagent_type: general-purpose) with prompt:
      "Run the patch-mode pipeline for one target. Working directory: <working_dir>. annotation-targets.json is already written. Run: targeted-intake-agent → applicable patch agents (token-system-builder, figma-instruction-writer) → design-validator. design-validator writes done/failed back to sharedPluginData. Agent specs: /Users/I750626/AI/agents/design-pipeline/"

4. Log: [design-pipe:listen] Dispatched N item(s) at <time>
```

### Step 3 — Write job ID

Write the CronCreate job ID to `<working_dir>/.listen_job_id`.

### Step 4 — Confirm

```
Listening for plugin instructions.
Working directory: <working_dir>
Polling every minute — submit from the Figma plugin panel to queue work.
Stop with /design-pipe:stop
```

## Rules

- Dispatch multiple pending items as parallel agents per tick
- Never mark items done — that is design-validator's responsibility
- If figma_execute fails (plugin not open), log a warning but keep the cron running
- Cron auto-expires after 3 days per Claude Code session limits
