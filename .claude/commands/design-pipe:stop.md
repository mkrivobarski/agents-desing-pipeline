Cancel the active `/design-pipe:listen` polling loop.

## Usage

```
/design-pipe:stop [working_dir]
```

## What you must do

1. Determine the working directory: use `$ARGUMENTS` if provided, otherwise `./pipeline-run`.

2. Read `<working_dir>/.listen_job_id` to get the stored cron job ID.

3. Call CronList to see all active jobs.

4. If a matching job ID is found, call CronDelete with that ID.
   If no stored ID but CronList shows a job whose prompt contains "design-pipe:listen" or "Design pipeline listen tick", cancel that one.

5. Call `figma_execute` to clear the listen indicator in the plugin (best-effort — skip if plugin not open):
   ```javascript
   figma.ui.postMessage({ type: 'PIPELINE_LISTEN_STATUS', data: { state: 'hidden' } });
   ```

6. Output:
   ```
   Pipeline listen mode stopped.
   ```
   Or if no active job was found:
   ```
   No active listen session found.
   ```

## Rules

- Only cancel jobs matching the pipeline listen pattern — do not cancel unrelated cron jobs
