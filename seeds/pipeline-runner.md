# Design Production Pipeline — Runner Guide

This document explains how to execute the 8-stage design production pipeline using the
Ouroboros `execute_seed` tool, how to pass outputs between stages, how to handle failures,
and the complete pipeline as pseudocode.

---

## Pipeline Overview

```
[Brief / Brand Docs]
        |
        v
Stage 1: seed-requirements-intake      --> requirements.json
        |
        v
Stage 2: seed-user-flows               --> user-flows.json
        |
        v
Stage 3: seed-wireframe-blueprints     --> screen-blueprints.json
        |
        v
Stage 4: seed-journey-map              --> journey-map.json + Figma "Journey Maps" page
        |
        v ── GATE 0: structure validated before token investment ──
        v
Stage 5: seed-lo-fi-wireframes         --> lo-fi-frames/index.json + Figma "Lo-Fi Wireframes" page
        |
        v
Stage 6: seed-token-system             --> token-map.json + Figma Variable collections
        |
        v ── GATE 1: token coverage ≥ 90% ──
        v
Stage 7: seed-component-build-plan     --> component-manifest.json + component-build-plan.json
        |
        v ── GATE 2: build plan reviewed ──
        v
Stage 8: seed-component-library        --> built-component-library.json + Figma COMPONENT nodes
        |
        v ── GATE 3: first-screen screenshot verified ──
        v
Stage 9: organism-composer runs (no seed needed — pure JSON orchestration)
          --> organism-manifest.json
        |
        v
Stage 10: seed-figma-instructions      --> figma-scripts/*.js
        |
        v (run figma_execute for each screen)
        v
Stage 11: seed-design-validation       --> validation-reports/[screen_id]__report.json (per screen)
        |
        v ── GATE 4: parity score ≥ 0.95 ──
        v
Stage 12: seed-delivery-package        --> delivery-package.json + handoff-summary.md
```

Stages 1–10 are strictly sequential: each stage depends on the output of the previous.
Stage 11 runs once per screen (parallelisable across screens). Stage 12 runs after all
Stage 11 reports are available and passing.

---

## Running Each Stage

### Stage 1 — Requirements Intake

**Input:** Design brief (file path, PDF path, or URL) + `requirements.schema.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-requirements-intake.yaml"),
  context = {
    brief_path: "/path/to/design-brief.pdf",
    schema_path: "/path/to/requirements.schema.json",
    brand_docs_path: "/path/to/brand-guidelines.pdf"  // optional
  }
)
```

**Output written to working directory:** `requirements.json`

**Check before proceeding:** Confirm `requirements.json` passes schema validation.
The seed's own acceptance criteria enforce this, but you can verify manually:

```bash
jsonschema --instance requirements.json requirements.schema.json
```

---

### Stage 2 — User Flows

**Input:** `requirements.json` from Stage 1 + `user-flows.schema.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-user-flows.yaml"),
  context = {
    requirements_path: "/working-dir/requirements.json",
    schema_path: "/path/to/user-flows.schema.json"
  }
)
```

**Output written to working directory:** `user-flows.json`

**Check before proceeding:** Confirm every screen slug from `requirements.json` appears
as a node id in `user-flows.json`.

---

### Stage 3 — Wireframe Blueprints

**Input:** `user-flows.json` + `requirements.json` + `screen-blueprints.schema.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-wireframe-blueprints.yaml"),
  context = {
    user_flows_path: "/working-dir/user-flows.json",
    requirements_path: "/working-dir/requirements.json",
    schema_path: "/path/to/screen-blueprints.schema.json"
  }
)
```

**Output written to working directory:** `screen-blueprints.json`

**Check before proceeding:** Confirm every screen id from `user-flows.json` has a
blueprint, and no slot uses a non-vocabulary type.

---

### Stage 4 — Journey Map

**Input:** `requirements.json` + `user-flows.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-journey-map.yaml"),
  context = {
    requirements_path: "/working-dir/requirements.json",
    user_flows_path: "/working-dir/user-flows.json"
  }
)
```

**Output written to working directory:** `journey-map.json`, Figma "Journey Maps" page

**Check before proceeding (GATE 0):** Review the journey map visually. Confirm that the
emotion arc is plausible and all key personas are covered. This is the last human
review point before token/component investment begins.

---

### Stage 5 — Lo-Fi Wireframes

**Input:** `screen-blueprints.json` + `user-flows.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-lo-fi-wireframes.yaml"),
  context = {
    blueprints_path: "/working-dir/screen-blueprints.json",
    user_flows_path: "/working-dir/user-flows.json"
  }
)
```

**Output written to working directory:** `lo-fi-frames/index.json`, Figma "Lo-Fi Wireframes" page

**Check before proceeding:** Review wireframes in Figma to validate layout hierarchy and
zone structure. Correct `screen-blueprints.json` if any structural issues are found —
it is far cheaper to fix blueprints now than after tokens and components are built.

---

### Stage 6 — Token System Build

**Input:** `screen-blueprints.json` + `requirements.json` + theme file(s)

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-token-system.yaml"),
  context = {
    blueprints_path: "/working-dir/screen-blueprints.json",
    requirements_path: "/working-dir/requirements.json",
    theme_path: "/path/to/design-tokens.json"  // optional
  }
)
```

**Output written to working directory:** `token-map.json`
**Output committed to Figma:** Primitives, Semantic, and Component variable collections

**Check before proceeding (GATE 1):** `coverage_score` in `token-map.json` must be ≥ 90.0.
If below 90%, review the `missing_tokens` array and either update the theme file or
extend the Component collection. Also check `contrast_failures` — all WCAG AA failures
must be resolved before proceeding.

---

### Stage 7 — Component Architecture

**Input:** `screen-blueprints.json` + `token-map.json` + `requirements.json` + component library inventory

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-component-build-plan.yaml"),
  context = {
    blueprints_path: "/working-dir/screen-blueprints.json",
    token_map_path: "/working-dir/token-map.json",
    requirements_path: "/working-dir/requirements.json",
    library_path: "/path/to/component-library.json"  // optional
  }
)
```

**Output written to working directory:** `component-manifest.json`, `component-build-plan.json`

**Check before proceeding (GATE 2):** Review `component-build-plan.json`'s `build_order`.
Confirm tier assignments are correct (atoms before molecules before organisms). Review any
`source: "build_required"` entries — these are gaps that `component-builder` will fill.

---

### Stage 8 — Component Library Build

**Input:** `component-build-plan.json` + `token-map.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-component-library.yaml"),
  context = {
    build_plan_path: "/working-dir/component-build-plan.json",
    token_map_path: "/working-dir/token-map.json"
  }
)
```

**Output written to working directory:** `built-component-library.json`
**Output committed to Figma:** All COMPONENT and COMPONENT_SET nodes on "Component Library" page

**Check before proceeding (GATE 3):** Review `build_warnings` in `built-component-library.json`.
Any entry with a non-empty `fallbacks` array has a token that could not be variable-bound.
Capture a screenshot of the first organism to visually verify component quality before
proceeding to screen assembly.

---

### Stage 9 — Figma Scripts

**Input:** `organism-manifest.json` + `built-component-library.json` + `component-manifest.json` + `token-map.json` + `screen-blueprints.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-figma-instructions.yaml"),
  context = {
    organism_manifest_path: "/working-dir/organism-manifest.json",
    library_path: "/working-dir/built-component-library.json",
    manifest_path: "/working-dir/component-manifest.json",
    token_map_path: "/working-dir/token-map.json",
    blueprints_path: "/working-dir/screen-blueprints.json"
  }
)
```

**Output written to working directory:** `figma-scripts/[screen_id].js` (one per screen), `figma-scripts/index.json`

**Before running in Figma:** Check that `organism-manifest.json` has no entries in `gaps[]`. Any gap means a required component was not built — rerun `seed-component-library.yaml` to address it before executing screen scripts.

**Running in Figma:** For each screen script in `figma-scripts/`, pass it to `figma_execute`. If a script returns `{ success: false }` with an ESCALATION error, do not proceed — rerun `seed-component-library.yaml` to rebuild the missing component node.

---

### Stage 10 — Organism Composer (no seed required)

The `organism-composer` agent reads `built-component-library.json`, `component-manifest.json`, and `screen-blueprints.json` and writes `organism-manifest.json`. This is pure JSON orchestration with no Figma calls — run it as a direct agent invocation:

```
Invoke: organism-composer agent
Input:  working-dir/built-component-library.json
        working-dir/component-manifest.json
        working-dir/screen-blueprints.json
Output: working-dir/organism-manifest.json
```

**Check before proceeding:** Review `organism-manifest.json`'s `gaps[]` array. Any gap is a `component-builder` failure — rerun `seed-component-library.yaml` to address missing organisms before continuing to screen script generation.

---

### Stage 11 — Design Validation

Run once per screen after executing its Figma instruction script.

**Input:** Screenshot of the Figma-built screen (captured via `figma_capture_screenshot`
or `figma_get_component_image`) + `screen-blueprints.json` + `token-map.json`
+ `requirements.json` + `validation-report.schema.json`

```
screenshot = figma_capture_screenshot(nodeId = "<screen-frame-node-id>")

ouroboros execute_seed(
  seed_content = file("seeds/seed-design-validation.yaml"),
  context = {
    screenshot: screenshot.url_or_base64,
    screen_id: "home-feed",
    blueprints_path: "/working-dir/screen-blueprints.json",
    token_map_path: "/working-dir/token-map.json",
    requirements_path: "/working-dir/requirements.json",
    iteration_number: 1,
    schema_path: "/path/to/validation-report.schema.json"
  }
)
```

**Output written to working directory:** `validation-reports/[screen_id]__default__report.json`

**Check before proceeding to Stage 12 (GATE 4):** All screens must have `parity_score >= 0.95`.
Screens failing after iteration 3 escalate to human review — do not proceed to delivery
until all screens pass or are explicitly accepted with documented deviations.

---

### Stage 12 — Delivery Package

**Input:** All `validation-reports/*.json` files + `built-component-library.json` + `organism-manifest.json` + `component-manifest.json` + `user-flows.json` + `token-map.json`

```
ouroboros execute_seed(
  seed_content = file("seeds/seed-delivery-package.yaml"),
  context = {
    validation_reports_dir: "/working-dir/validation-reports/",
    library_path: "/working-dir/built-component-library.json",
    organism_manifest_path: "/working-dir/organism-manifest.json",
    manifest_path: "/working-dir/component-manifest.json",
    user_flows_path: "/working-dir/user-flows.json",
    token_map_path: "/working-dir/token-map.json"
  }
)
```

**Output written to working directory:** `delivery-package.json` + `handoff-summary.md`

---

## Passing Outputs Between Stages

Each seed reads its inputs from the working directory using the file paths provided
in the execution context. The simplest approach is to use a single working directory
for all stages in a pipeline run:

```
/project/runs/run-2026-03-13/
  requirements.json               (Stage 1 output)
  user-flows.json                 (Stage 2 output)
  screen-blueprints.json          (Stage 3 output)
  journey-map.json                (Stage 4 output)
  lo-fi-frames/index.json         (Stage 5 output)
  token-map.json                  (Stage 6 output)
  component-manifest.json         (Stage 7 output)
  component-build-plan.json       (Stage 7 output)
  built-component-library.json    (Stage 8 output)
  organism-manifest.json          (Stage 10 output)
  figma-scripts/                  (Stage 9 output)
  validation-reports/             (Stage 11 outputs, one per screen)
  delivery-package.json           (Stage 12 output)
  handoff-summary.md              (Stage 12 output)
```

When calling `execute_seed`, always pass the run directory as the working directory
so all stages can locate their inputs via relative paths within it.

For Ouroboros sessions, pass a `session_id` to chain runs:

```
session = ouroboros_execute_seed(seed="stage1...", session_id="run-2026-03-13")
// reuse session_id for stages 2–8 to share memory/context
ouroboros_execute_seed(seed="stage2...", session_id="run-2026-03-13")
```

---

## Handling Failures at Each Stage

### Stage 1 Failure — Requirements Intake

**Symptom:** `requirements.json` fails schema validation, or fewer than 3 screens are found.

**Fix:** Review the design brief for completeness. Common issues:
- Brief does not explicitly name screens → add a "Screens" section to the brief
- Color values are not specified → add a color palette section to the brief or
  provide a brand-guidelines document as additional context
- No user flows described → add a "User Journeys" section

**Action:** Rerun Stage 1 with the updated brief. No downstream stages need to be rerun
until Stage 1 produces a valid `requirements.json`.

---

### Stage 2 Failure — User Flows

**Symptom:** `user-flows.json` fails schema validation, or screens from requirements are
missing as nodes, or orphaned nodes exist.

**Fix:**
- Missing nodes → check `requirements.json` screen slugs match the node ids; if
  `requirements.json` used different naming, rerun Stage 1 with clearer screen names
- Orphaned nodes → add at least one entry edge; check if a flow was omitted
- No error state path → explicitly ask the agent to model at least one error path

**Action:** Rerun Stage 2 only. `requirements.json` does not need to change unless the
underlying screen inventory was wrong.

---

### Stage 3 Failure — Wireframe Blueprints

**Symptom:** `screen-blueprints.json` fails validation, invalid slot types, or a screen
from `user-flows.json` has no blueprint.

**Fix:**
- Invalid slot types → the agent used a non-vocabulary name; rerun with an explicit
  instruction to use only the vocabulary list
- Missing screen blueprint → the agent may have skipped modal or overlay screens;
  rerun with an explicit instruction to blueprint all node types including overlays
- Missing async state blueprints → rerun with explicit instruction to produce
  loading/error/empty blueprints for all screens with data_binding slots

**Action:** Rerun Stage 3 only.

---

### Stage 4 Failure — Journey Map

**Symptom:** Journey map missing personas, emotion arc is implausible, or Figma page was not created.

**Fix:**
- Missing personas → check `requirements.json` for persona definitions; rerun with additional context
- Emotion arc doesn't match user flows → verify `user-flows.json` is complete and accurate first

**Action:** Rerun Stage 4 only.

---

### Stage 5 Failure — Lo-Fi Wireframes

**Symptom:** Wireframes missing screens, zone structure is wrong, or Figma page was not created.

**Fix:**
- Missing screens → check `screen-blueprints.json` has entries for all screens in `user-flows.json`
- Wrong zone structure → fix `screen-blueprints.json` and rerun Stage 5

**Action:** Rerun Stage 5 only. If the blueprint was the root cause, fix `screen-blueprints.json` first.

---

### Stage 6 Failure — Token System Build

**Symptom:** `token-map.json` has `coverage_score < 90`, missing `variable_id` fields, or Figma Variables were not committed.

**Fix:**
- Coverage below 90% → review `missing_tokens` in `token-map.json`; extend theme file or add custom tokens
- Missing `variable_id` → the agent failed to commit variables; check `figma_batch_create_variables` errors in the run
- Contrast failures → update Semantic color aliases to pass WCAG AA (4.5:1 ratio)

**Action:** Rerun Stage 6 only (possibly after updating the theme file).

---

### Stage 7 Failure — Component Architecture

**Symptom:** `component-build-plan.json` has invalid build order (molecule before its atom deps), or `component-manifest.json` has slots missing.

**Fix:**
- Invalid build order → manually inspect `children` arrays; a molecule depending on an atom that comes later in `build_order` is the violation
- Missing slots → check `screen-blueprints.json` slots are all covered in `component-manifest.json`

**Action:** Rerun Stage 7 only.

---

### Stage 8 Failure — Component Library Build

**Symptom:** `built-component-library.json` has `build_warnings` with errors, components have non-empty `fallbacks` arrays, or Figma COMPONENT nodes are missing.

**Fix:**
- Build errors → review specific error in `build_warnings`; common issues: font not loaded, `combineAsVariants` called before appending to section
- Token fallbacks → the token exists in `component-build-plan.json` but has no `variable_id` in `token-map.json`; rerun Stage 6 first
- Wrong post-combine IDs → the agent must read IDs from `buttonSet.children` after `combineAsVariants`, not from the pre-combine components

**Action:** Rerun Stage 8. If token binding is the root cause, fix Stage 6 first.

---

### Stage 9 Failure — Figma Instructions

**Symptom:** Scripts return ESCALATION errors, or `organism-manifest.json` has gaps blocking script generation.

**Fix — ESCALATION (node not found):**
  The component node ID stored in `built-component-library.json` is no longer valid. Rerun Stage 8 to rebuild the affected component. The new node IDs will be recorded in `built-component-library.json`, then rerun Stage 9.

**Fix — organism manifest gaps:**
  An organism required by a screen was not built. Check `organism-manifest.json`'s `gaps[]`, identify the missing component, and rerun Stage 8.

**Fix — color object with alpha:**
  The script emitted `{ r, g, b, a }`. Rerun Stage 9 with an explicit constraint reminder that alpha must be set via `opacity` on the paint object, not in the color object.

**Fix — direct page assignment:**
  Rerun with explicit instruction: "Use `await figma.setCurrentPageAsync(page)` exclusively."

**Action:** Rerun Stage 9. If node IDs are stale, rerun Stage 8 first.

---

### Stage 11 Failure — Design Validation

**Symptom:** Validation report shows `parity_score < 0.95`. Fix-iterate loop:

**Iteration 1 failure:**
  Review `blocking_fixes` in the report. Check the `instance` category first — wrong `mainComponentId` means the wrong component was instantiated and requires a Stage 9 rerun (not just a property fix).
  For spacing/color/typography issues, apply `fix_instruction.figma_api_hint` corrections via `figma_execute`. Re-capture screenshot and rerun Stage 11 with `iteration_number: 2`.

**Iteration 2 failure:**
  Repeat above. Apply remaining fixes. Rerun with `iteration_number: 3`.

**Iteration 3 failure (escalation required):**
  Do not proceed to Stage 12 for this screen. Options:
  - Manually fix in Figma and run a Stage 11 recheck with `iteration_number: 1`
    (reset iteration counter for the manual fix round)
  - Exclude the screen from the delivery package (it will appear in `screens_excluded`)
  - Raise the issue with the design owner for a spec change, then rerun from Stage 3
    for that screen only

**Action:** Rerun Stage 11 per screen. Stages 1–10 do not need to rerun unless the
spec itself changes.

---

### Stage 12 Failure — Delivery Package

**Symptom:** `delivery-package.json` fails validation, `overall_coverage_pct <= 75`,
or `handoff_summary_md` is missing.

**Fix:**
- Coverage below 75% → return to Stage 6, improve token coverage, and rerun stages 6–11
- Missing screens → check that all validation reports exist and have `parity_score >= 0.95`
- Screen order wrong → verify `user-flows.json` has correct entry points and edges;
  if user flows changed, rerun Stage 2 and all downstream stages
- Missing handoff_summary_md → rerun Stage 12 with explicit instruction to write the
  markdown summary field

**Action:** Rerun Stage 12 only, unless the root cause is in an upstream artifact.

---

## Complete Pipeline Pseudocode

```bash
#!/usr/bin/env bash
# Design Production Pipeline — Full Run
# Usage: ./run-pipeline.sh <brief_path> <run_id>

set -euo pipefail

BRIEF="$1"
RUN_ID="${2:-run-$(date +%Y%m%d-%H%M%S)}"
WORKDIR="/project/runs/$RUN_ID"
SEEDS="/path/to/seeds"
SCHEMAS="/path/to/schemas"
SESSION_ID="$RUN_ID"

mkdir -p "$WORKDIR"
echo "Pipeline run: $RUN_ID"
echo "Working dir:  $WORKDIR"

# ─────────────────────────────────────────────
# STAGE 1: Requirements Intake
# ─────────────────────────────────────────────
echo "[Stage 1] Parsing design brief..."

ouroboros execute_seed \
  --seed "$SEEDS/seed-requirements-intake.yaml" \
  --session "$SESSION_ID" \
  --context brief_path="$BRIEF" \
  --context schema_path="$SCHEMAS/requirements.schema.json" \
  --workdir "$WORKDIR"

# Validate output
jsonschema --instance "$WORKDIR/requirements.json" "$SCHEMAS/requirements.schema.json" \
  || { echo "FAIL Stage 1: requirements.json invalid"; exit 1; }

SCREEN_COUNT=$(jq '.screens | length' "$WORKDIR/requirements.json")
[ "$SCREEN_COUNT" -ge 3 ] \
  || { echo "FAIL Stage 1: fewer than 3 screens ($SCREEN_COUNT found)"; exit 1; }

echo "[Stage 1] PASS — $SCREEN_COUNT screens found"

# ─────────────────────────────────────────────
# STAGE 2: User Flows
# ─────────────────────────────────────────────
echo "[Stage 2] Building user flows..."

ouroboros execute_seed \
  --seed "$SEEDS/seed-user-flows.yaml" \
  --session "$SESSION_ID" \
  --context requirements_path="$WORKDIR/requirements.json" \
  --context schema_path="$SCHEMAS/user-flows.schema.json" \
  --workdir "$WORKDIR"

jsonschema --instance "$WORKDIR/user-flows.json" "$SCHEMAS/user-flows.schema.json" \
  || { echo "FAIL Stage 2: user-flows.json invalid"; exit 1; }

echo "[Stage 2] PASS"

# ─────────────────────────────────────────────
# STAGE 3: Wireframe Blueprints
# ─────────────────────────────────────────────
echo "[Stage 3] Generating wireframe blueprints..."

ouroboros execute_seed \
  --seed "$SEEDS/seed-wireframe-blueprints.yaml" \
  --session "$SESSION_ID" \
  --context user_flows_path="$WORKDIR/user-flows.json" \
  --context requirements_path="$WORKDIR/requirements.json" \
  --context schema_path="$SCHEMAS/screen-blueprints.schema.json" \
  --workdir "$WORKDIR"

jsonschema --instance "$WORKDIR/screen-blueprints.json" "$SCHEMAS/screen-blueprints.schema.json" \
  || { echo "FAIL Stage 3: screen-blueprints.json invalid"; exit 1; }

echo "[Stage 3] PASS"

# ─────────────────────────────────────────────
# STAGE 4: Token Mapping
# ─────────────────────────────────────────────
echo "[Stage 4] Mapping design tokens..."

ouroboros execute_seed \
  --seed "$SEEDS/seed-token-mapping.yaml" \
  --session "$SESSION_ID" \
  --context blueprints_path="$WORKDIR/screen-blueprints.json" \
  --context theme_path="/path/to/design-tokens.json" \
  --context schema_path="$SCHEMAS/token-map.schema.json" \
  --workdir "$WORKDIR"

jsonschema --instance "$WORKDIR/token-map.json" "$SCHEMAS/token-map.schema.json" \
  || { echo "FAIL Stage 4: token-map.json invalid"; exit 1; }

COVERAGE=$(jq '.coverage_score' "$WORKDIR/token-map.json")
COVERAGE_OK=$(echo "$COVERAGE >= 80" | bc -l)
[ "$COVERAGE_OK" -eq 1 ] \
  || { echo "FAIL Stage 4: coverage_score=$COVERAGE is below 80%"; exit 1; }

echo "[Stage 4] PASS — token coverage: $COVERAGE%"

# ─────────────────────────────────────────────
# STAGE 5: Component Manifest
# ─────────────────────────────────────────────
echo "[Stage 5] Resolving component manifest..."

ouroboros execute_seed \
  --seed "$SEEDS/seed-component-manifest.yaml" \
  --session "$SESSION_ID" \
  --context blueprints_path="$WORKDIR/screen-blueprints.json" \
  --context token_map_path="$WORKDIR/token-map.json" \
  --context library_path="/path/to/component-library.json" \
  --context schema_path="$SCHEMAS/component-manifest.schema.json" \
  --workdir "$WORKDIR"

jsonschema --instance "$WORKDIR/component-manifest.json" "$SCHEMAS/component-manifest.schema.json" \
  || { echo "FAIL Stage 5: component-manifest.json invalid"; exit 1; }

echo "[Stage 5] PASS"

# ─────────────────────────────────────────────
# STAGE 6: Figma Instructions
# ─────────────────────────────────────────────
echo "[Stage 6] Generating Figma scripts..."

ouroboros execute_seed \
  --seed "$SEEDS/seed-figma-instructions.yaml" \
  --session "$SESSION_ID" \
  --context manifest_path="$WORKDIR/component-manifest.json" \
  --context token_map_path="$WORKDIR/token-map.json" \
  --context blueprints_path="$WORKDIR/screen-blueprints.json" \
  --context schema_path="$SCHEMAS/figma-instructions.schema.json" \
  --workdir "$WORKDIR"

# Syntax-check all generated scripts
SYNTAX_ERRORS=0
for SCREEN in $(jq -r 'keys[]' "$WORKDIR/figma-instructions.json"); do
  jq -r --arg s "$SCREEN" '.[$s]' "$WORKDIR/figma-instructions.json" \
    | node --check /dev/stdin 2>&1 \
    && echo "  syntax OK: $SCREEN" \
    || { echo "  syntax FAIL: $SCREEN"; SYNTAX_ERRORS=$((SYNTAX_ERRORS + 1)); }
done

[ "$SYNTAX_ERRORS" -eq 0 ] \
  || { echo "FAIL Stage 6: $SYNTAX_ERRORS scripts have syntax errors"; exit 1; }

echo "[Stage 6] PASS — all scripts syntax-valid"

# ─────────────────────────────────────────────
# STAGE 6b: Execute Scripts in Figma
# (Run manually or via figma_execute for each screen)
# ─────────────────────────────────────────────
echo "[Stage 6b] Run each screen script in Figma via figma_execute:"
echo "  figma_execute(code = jq '.[\"<screen-slug>\"]' $WORKDIR/figma-instructions.json)"
echo "  Capture the resulting frame node IDs for Stage 7."
echo "  (Pausing here — continue when Figma execution is complete)"
# In a real automated run, this would call figma_execute for each screen
# and record the resulting node IDs.

# ─────────────────────────────────────────────
# STAGE 7: Design Validation (per screen)
# ─────────────────────────────────────────────
echo "[Stage 7] Validating screens..."

VALIDATION_FAILURES=0
for SCREEN in $(jq -r 'keys[]' "$WORKDIR/figma-instructions.json"); do
  echo "  Validating screen: $SCREEN"

  # In practice, capture screenshot via figma_capture_screenshot(nodeId=<id>)
  SCREENSHOT="<screenshot_url_or_base64_for_$SCREEN>"

  for ITERATION in 1 2 3; do
    ouroboros execute_seed \
      --seed "$SEEDS/seed-design-validation.yaml" \
      --session "$SESSION_ID" \
      --context screen_id="$SCREEN" \
      --context screenshot="$SCREENSHOT" \
      --context blueprints_path="$WORKDIR/screen-blueprints.json" \
      --context token_map_path="$WORKDIR/token-map.json" \
      --context requirements_path="$WORKDIR/requirements.json" \
      --context iteration_number="$ITERATION" \
      --context schema_path="$SCHEMAS/validation-report.schema.json" \
      --workdir "$WORKDIR"

    REPORT="$WORKDIR/validation-report-$SCREEN.json"
    SCORE=$(jq '.summary.overall_score' "$REPORT")
    SCORE_OK=$(echo "$SCORE >= 0.8" | bc -l)

    if [ "$SCORE_OK" -eq 1 ]; then
      echo "    PASS (iteration $ITERATION, score=$SCORE)"
      break
    else
      echo "    FAIL (iteration $ITERATION, score=$SCORE)"
      if [ "$ITERATION" -eq 3 ]; then
        echo "    ESCALATION REQUIRED for screen: $SCREEN"
        VALIDATION_FAILURES=$((VALIDATION_FAILURES + 1))
      else
        # Apply fix instructions from the report, re-capture screenshot
        echo "    Applying fixes and re-capturing screenshot..."
        SCREENSHOT="<updated_screenshot_for_$SCREEN>"
      fi
    fi
  done
done

[ "$VALIDATION_FAILURES" -eq 0 ] \
  || echo "WARNING: $VALIDATION_FAILURES screen(s) require manual escalation before Stage 8"

echo "[Stage 7] Complete"

# ─────────────────────────────────────────────
# STAGE 8: Delivery Package
# ─────────────────────────────────────────────
echo "[Stage 8] Assembling delivery package..."

ouroboros execute_seed \
  --seed "$SEEDS/seed-delivery-package.yaml" \
  --session "$SESSION_ID" \
  --context validation_reports_dir="$WORKDIR" \
  --context manifest_path="$WORKDIR/component-manifest.json" \
  --context user_flows_path="$WORKDIR/user-flows.json" \
  --context token_map_path="$WORKDIR/token-map.json" \
  --context schema_path="$SCHEMAS/delivery-package.schema.json" \
  --workdir "$WORKDIR"

jsonschema --instance "$WORKDIR/delivery-package.json" "$SCHEMAS/delivery-package.schema.json" \
  || { echo "FAIL Stage 8: delivery-package.json invalid"; exit 1; }

FINAL_COVERAGE=$(jq '.token_coverage_report.overall_coverage_pct' "$WORKDIR/delivery-package.json")
echo "[Stage 8] PASS — final token coverage: $FINAL_COVERAGE%"

echo ""
echo "Pipeline complete. Outputs in: $WORKDIR"
echo "  Delivery package: $WORKDIR/delivery-package.json"
echo "  Handoff summary:  $WORKDIR/handoff-summary.md"
```

---

## Quick Reference — Failure Recovery Decision Tree

```
Stage failed?
  |
  +-- Stage 1  → Fix the design brief → Rerun Stage 1 → Continue from Stage 2
  |
  +-- Stage 2  → Fix requirements.json (if screen inventory was wrong) OR
  |              Rerun Stage 2 with clearer instructions → Continue from Stage 3
  |
  +-- Stage 3  → Rerun Stage 3 only → Continue from Stage 4
  |
  +-- Stage 4  → Fix requirements.json or user-flows.json → Rerun Stage 4 → Continue from Stage 5
  |
  +-- Stage 5  → Fix screen-blueprints.json → Rerun Stage 5 → Continue from Stage 6
  |
  +-- Stage 6  → Update theme file OR extend token definitions → Rerun Stage 6 → Continue from Stage 7
  |              (GATE 1: coverage must reach 90% before proceeding)
  |
  +-- Stage 7  → Fix component-build-plan.json build order → Rerun Stage 7 → Continue from Stage 8
  |
  +-- Stage 8  → Fix token bindings (if fallbacks) OR fix combineAsVariants pattern → Rerun Stage 8
  |              (GATE 3: verify first-screen screenshot before assembling screens)
  |
  +-- Stage 9  → If ESCALATION: rerun Stage 8 first to rebuild component nodes → Rerun Stage 9
  |
  +-- Stage 10 → If gaps: rerun Stage 8 for missing organisms → Rerun Stage 10
  |
  +-- Stage 11 → Apply fix_instructions in Figma → Re-capture screenshot → Rerun Stage 11
  |              (up to 3 iterations; escalate if all 3 fail)
  |              (instance category FAIL = wrong component node → rerun Stage 9)
  |
  +-- Stage 12 → Check upstream validation reports → Rerun Stage 12
```

---

## Notes on Session IDs

Using a consistent `session_id` across all stages in a pipeline run allows Ouroboros
to maintain shared context between seeds. The agent can reference decisions made in
earlier stages (e.g. screen names chosen in Stage 1, token names established in Stage 4)
without requiring explicit re-injection of those values into every seed context.

Recommended session ID format: `design-pipeline-{project-slug}-{YYYYMMDD}`

Example: `design-pipeline-checkout-flow-20260313`
