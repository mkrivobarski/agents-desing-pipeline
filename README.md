# Design Pipeline — Agent Reference

This folder contains the AI agent instructions that drive every stage of the pipeline.
You don't run these directly — they're invoked automatically when you use the slash commands
in Claude Code (`/design-pipe:new`, `/design-pipe:review`, etc.).

**This document is a reference for understanding what runs under the hood.**
For setup and usage instructions, see the [main README](../README.md).

---

# Design Pipeline — Claude Code Agent System

A multi-agent pipeline that transforms **Figma files, design briefs, or PRDs** into
validated, production-ready Figma screens with full prototype links, design-review
reports, consistency analysis, and handoff documentation.

The pipeline runs in two modes:
- **From Figma** — ingest an existing Figma file as source of truth, review it, extend it
- **From Brief** — start with a text/PDF brief and build screens from scratch

---

## Full Pipeline Map

```
╔══════════════════════════════════════════════════════════════════════╗
║  FIGMA-FIRST PATH          │   BRIEF-FIRST PATH                      ║
╠══════════════════════════════════════════════════════════════════════╣
║                             │                                         ║
║  [Figma File URL]           │   [Brief / PRD / Brand docs]            ║
║         │                   │              │                          ║
║         ▼                   │              ▼                          ║
║  figma-intake-agent         │   requirements-analyst                  ║
║  figma-frame-extractor      │         │                               ║
║  figma-token-extractor      │         ▼                               ║
║         │                   │   flow-architect                        ║
║         └─────────┬─────────┘         │                              ║
║                   │                   ▼                               ║
║                   │         wireframe-strategist                      ║
║                   │                   │                               ║
╚═══════════════════╪═══════════════════╪═══════════════════════════════╝
                    │                   │
                    ▼                   ▼
           ┌────────────────────────────────────┐
           │         SHARED PIPELINE CORE        │
           └────────────────────────────────────┘
                           │
              ┌────────────┼────────────────────┐
              │                                  │
              ▼                                  ▼
      journey-mapper               [Design Review Track]
   (journey-map.json)              design-review-agent
              │                    consistency-analyser
              ▼                               │
       lo-fi-builder                          │ (recommendations feed
  (lo-fi-frames/index.json)                   │  back into token and
       ── GATE 0 ──                           │  component decisions)
              │                               │
              ▼                               │
   creative-director ◄──────────────────────┘
  (creative-direction.json —
   colour, type, icons, shape,
   media, motion decisions)
       ── GATE 0.5 ──
              │
              ▼
   token-system-builder
     (token-map.json +
      Figma Variables committed)
       ── GATE 1 ──
              │
              ▼
    component-architect
  (component-manifest.json +
   component-build-plan.json)
       ── GATE 2 ──
              │
              ▼
    component-builder
  (built-component-library.json
   + Figma COMPONENT nodes)
       ── GATE 3 ──
              │
              ▼
    organism-composer
  (organism-manifest.json)
              │
              ▼
  figma-instruction-writer
   (figma-scripts/*.js)
              │
              ▼ [figma_execute each script]
              │
              ▼
      design-validator ──► validation-reports/
       ── GATE 4 ──
              │
         [parallel]
         ├── ux-evaluator ──► ux-evaluation-report.json
         └── brand-compliance-agent ──► brand-compliance-report.json
              │
              ▼
      prototype-linker ──► [Figma prototype with navigation links]
              │
              ▼
     delivery-sequencer ──► HANDOFF.md, TOKEN_COVERAGE.md, CODE_CONNECT.md
```

---

## Agents

### Figma Input Stage

| Agent | Input | Output |
|---|---|---|
| `figma-intake-agent` | Figma file URL or fileKey | `figma-source.json`, `requirements.json`, `component-library.json` |
| `figma-frame-extractor` | `figma-source.json` | `extracted-frames/[screen_id].json`, `extracted-frames/index.json` |
| `figma-token-extractor` | Figma variables + styles (via MCP) | `figma-tokens.json`, `figma-tokens.css`, `figma-tokens.ts` |

### Design Brief Stage (Brief-First path only)

| Agent | Input | Output |
|---|---|---|
| `requirements-analyst` | Brief text/PDF/URL, reference files/URLs | `requirements.json`, `reference-materials.json` |
| `flow-architect` | `requirements.json` | `user-flows.json`, `user-flows.mermaid` |
| `wireframe-strategist` | `requirements.json`, `user-flows.json` | `screen-blueprints.json` |

### Design Review Track (runs in parallel with core pipeline)

| Agent | Input | Output |
|---|---|---|
| `design-review-agent` | `extracted-frames/index.json`, `figma-tokens.json` | `design-review.json`, `design-review-report.md` |
| `consistency-analyser` | `extracted-frames/index.json`, `figma-tokens.json` | `consistency-report.json`, `consistency-report.md` |

**Design review checks:**
- Accessibility (contrast ratios, tap targets, focus order)
- Consistency (same semantic role, different styles)
- Token adherence (hardcoded values not in system)
- Spacing system (off-scale values)
- Typography hierarchy (too many sizes/weights)
- Pattern abstraction (recurring UI patterns → component candidates)
- Naming conventions (default layer names)
- Design debt (duplicates, hidden layers, detached components)
- Navigation coherence (interactive element coverage, back controls, tab bar presence, empty state completeness)

**Consistency analysis produces:**
- Colour frequency map (in-system vs rogue hex values)
- Spacing histogram (in-scale vs outliers)
- Typography usage table
- Component drift scores
- Icon family audit
- Overall consistency score (0–1)

### Core Build Pipeline

| Agent | Input | Output | Notes |
|---|---|---|---|
| `journey-mapper` | `requirements.json`, `user-flows.json` | `journey-map.json`, Figma "Journey Maps" page | First pipeline artifact |
| `lo-fi-builder` | `screen-blueprints.json`, `user-flows.json` | `lo-fi-frames/index.json`, Figma "Lo-Fi Wireframes" page | Grey-box structural wireframes |
| `creative-director` | `reference-materials.json`, `requirements.json`, `journey-map.json` | `creative-direction.json` | Makes all open visual decisions: palette, type, icons, shape, media, motion; downstream agents read this as authoritative source |
| `token-system-builder` | `creative-direction.json`, `screen-blueprints.json`, `requirements.json`, theme files | `token-map.json` + committed Figma Variable collections | Reads creative-direction for all colour, type, and radius values |
| `component-architect` | `screen-blueprints.json`, `token-map.json`, `requirements.json`, `creative-direction.json` | `component-manifest.json`, `component-build-plan.json` | Uses creative-direction for density, radius, and icon sizing |
| `component-builder` | `component-build-plan.json`, `token-map.json` | `built-component-library.json` + Figma COMPONENT nodes | Builds real Figma COMPONENT_SET nodes via `combineAsVariants` |
| `organism-composer` | `built-component-library.json`, `component-manifest.json`, `screen-blueprints.json` | `organism-manifest.json` | Placement manifest only; no Figma calls; no `parentId` |
| `copy-writer` | `organism-manifest.json`, `requirements.json`, `creative-direction.json`, `screen-blueprints.json`, `component-manifest.json` | `copy-manifest.json` + updated `organism-manifest.json` | Fills placeholder TEXT prop_overrides with semantically appropriate copy; never modifies layout, variants, or specified copy |
| `icon-mapper` | `organism-manifest.json`, `creative-direction.json`, `component-manifest.json`, `requirements.json`, `screen-blueprints.json` | `icon-manifest.json` + updated `organism-manifest.json` | Assigns semantically appropriate icon names to null INSTANCE_SWAP icon slots; never modifies layout, variants, text props, or specified icons |
| `image-mapper` | `organism-manifest.json`, `creative-direction.json`, `component-manifest.json`, `requirements.json`, `screen-blueprints.json` | `image-manifest.json` + updated `organism-manifest.json` | Detects image slots via slot_type, prop name signals, data bindings, and organism name heuristics; assigns content categories and aspect ratios |
| `icon-library-builder` | `icon-manifest.json`, `creative-direction.json`, `requirements.json` | `icon-library.json` | Locates or creates icon component nodes in Figma; writes icon name → nodeId map |
| `image-library-builder` | `image-manifest.json`, `creative-direction.json`, `token-map.json` | `image-library.json` | Creates styled placeholder rectangles in Figma on Assets page; brand-tinted fills, diagonal-cross marker, label text; writes image name → nodeId map |
| `figma-instruction-writer` | `organism-manifest.json`, `token-map.json`, `icon-library.json`, `image-library.json` | `figma-scripts/scene.json`, `figma-scripts/index.json` | Uses manifest_to_scene → compile_scene → execute_instructions → figma_execute; passes icon_map and image_map (when fill_images: true) |
| `design-validator` | Screenshots, `built-component-library.json`, blueprints, `organism-manifest.json`, token-map | `validation-reports/[screen_id]__report.json` | Depth-6 extraction; instance validation category |

### Gate 4 Evaluation Stage

| Agent | Input | Output | Notes |
|---|---|---|---|
| `ux-evaluator` | `ux-acceptance-brief.json`, `organism-manifest.json`, `built-component-library.json`, `validation-reports/`, `user-flows.json`, `lo-fi-frames/gate0-evaluation.json` | `ux-evaluation-report.json`, `ux-evaluation-report.md` | Confirmation-mode; runs in parallel with brand-compliance-agent after design-validator |
| `brand-compliance-agent` | `requirements.json`, `extracted-frames/index.json`, `figma-tokens.json`, `ux-acceptance-brief.json`, `desirability-brief.json` | `brand-compliance-report.json`, `brand-compliance-report.md` | Includes Emotional Tone category; runs in parallel with ux-evaluator after design-validator |

### Output Stage

| Agent | Input | Output |
|---|---|---|
| `prototype-linker` | `user-flows.json`, `figma-source.json` | `prototype-links.json` + Figma prototype reactions |
| `delivery-sequencer` | All pipeline artifacts | `delivery-package.json`, `HANDOFF.md`, `TOKEN_COVERAGE.md`, `CODE_CONNECT.md` |

---

## Prototype Linker

The `prototype-linker` agent uses `figma_execute` to set native Figma prototype
reactions on every frame, based on edges in `user-flows.json`.

**Trigger mapping:**

| Flow edge trigger | Figma reaction type | Transition |
|---|---|---|
| `tap` / `click` | `ON_CLICK` | SMART_ANIMATE (0.3 s EASE_OUT) |
| `hover` | `ON_HOVER` | SMART_ANIMATE |
| `press` | `ON_PRESS` | SMART_ANIMATE |
| `drag` / `swipe` | `DRAG` | SMART_ANIMATE |
| `auto` / `timeout` | `AFTER_TIMEOUT` (2 000 ms) | DISSOLVE (0.4 s) |
| `back` | `ON_BACK_GESTURE` | BACK action |
| `overlay` edges | `ON_CLICK` | MOVE_IN |

Entry-point frames are registered as `page.flowStartingPoints` so the Figma
prototype player presents named flows.

---

## Ouroboros Seeds

Each pipeline stage has a corresponding Ouroboros seed in `seeds/`:

| Seed | Stage |
|---|---|
| `seed-figma-intake.yaml` | Figma file ingestion |
| `seed-requirements-intake.yaml` | Brief parsing |
| `seed-user-flows.yaml` | User flow generation |
| `seed-wireframe-blueprints.yaml` | Wireframe blueprints |
| `seed-journey-map.yaml` | Journey map + emotion analysis |
| `seed-lo-fi-wireframes.yaml` | Lo-fi structural wireframes |
| `seed-creative-direction.yaml` | Creative direction — all open visual decisions |
| `seed-token-system.yaml` | Token system build + Figma Variable commit |
| `seed-component-build-plan.yaml` | Component architecture + build plan |
| `seed-component-library.yaml` | Component library build (atoms → molecules → organisms) |
| `seed-figma-instructions.yaml` | Figma script generation from organism manifest |
| `seed-design-validation.yaml` | Validation loop |
| `seed-design-review.yaml` | Design review |
| `seed-consistency-analysis.yaml` | Consistency analysis |
| `seed-prototype-linking.yaml` | Prototype link creation |
| `seed-delivery-package.yaml` | Final delivery package |

Run any seed with:
```
ooo run path/to/seed.yaml
```

These seeds are used internally by the pipeline and by the Ouroboros workflow engine.
If you're not using Ouroboros, you don't need to interact with them directly.

---

## Figma MCP Tools Used

| Tool | Used by |
|---|---|
| `figma_get_file_data` | figma-intake-agent, prototype-linker |
| `figma_get_design_system_kit` | figma-intake-agent, figma-token-extractor |
| `figma_get_variables` | figma-token-extractor, token-system-builder (merge check) |
| `figma_get_styles` | figma-token-extractor |
| `figma_get_component_for_development` | figma-frame-extractor, design-review-agent |
| `figma_get_component` | figma-frame-extractor, design-review-agent |
| `figma_get_component_image` | figma-frame-extractor, design-review-agent |
| `figma_capture_screenshot` | design-review-agent, design-validator |
| `figma_search_components` | component-architect (live discovery) |
| `figma_get_component_details` | component-architect (live discovery) |
| `figma_create_variable_collection` | token-system-builder |
| `figma_batch_create_variables` | token-system-builder |
| `figma_batch_update_variables` | token-system-builder |
| `figma_execute` | lo-fi-builder (wireframes), component-builder (COMPONENT nodes), figma-instruction-writer (screens via execute_instructions JS), prototype-linker (reactions), design-validator (ground truth extraction), targeted-intake-agent (snapshots) |
| `figma_take_screenshot` | prototype-linker (post-link validation), figma-instruction-writer (post-render verification) |
| `figma_get_comments` | design-validator (optional) |
| `figma_post_comment` | design-validator (optional — posts inline fix notes) |
| `mcp__semantic-figma__manifest_to_scene` | figma-instruction-writer |
| `mcp__semantic-figma__compile_scene` | figma-instruction-writer |
| `mcp__semantic-figma__execute_instructions` | figma-instruction-writer |
| `mcp__semantic-figma__read_back_scene` | design-validator (Track 1 structural diff) |
| `mcp__semantic-figma__update_scene` | targeted-intake-agent (Phase 3b scene delta for non-structural patches) |

---

## Data Flow

```
figma-source.json          ← figma-intake-agent
    │
    ├─► extracted-frames/  ← figma-frame-extractor
    │       │
    │       └─► design-review-agent ──► design-review.json
    │           consistency-analyser ─► consistency-report.json
    │
    └─► figma-tokens.json  ← figma-token-extractor
            │
requirements.json ─────────┤
user-flows.json ───────────┤
screen-blueprints.json ────┤
            │
            ├─► ux-acceptance-brief.json (stub)  ← pipeline-intake-agent
            │       │
            ├─► journey-map.json          ← journey-mapper
            │   ux-acceptance-brief.json (screens[] populated)
            │
            ├─► screen-blueprints.json    ← wireframe-strategist
            │   ux-acceptance-brief.json (cta slot + cognitive count enriched)
            │
            ├─► lo-fi-frames/index.json   ← lo-fi-builder
            │   lo-fi-frames/gate0-evaluation.json
            │
            ├─► creative-direction.json   ← creative-director
            │   desirability-brief.json
            │
            ▼
        token-map.json  ← token-system-builder
        (+ Figma Variable collections committed)
            │
            ▼
     component-manifest.json   ← component-architect
     component-build-plan.json
            │
            ▼
     built-component-library.json  ← component-builder
     (+ Figma COMPONENT nodes built)
            │
            ▼
     organism-manifest.json  ← organism-composer
     copy-manifest.json      ← copy-writer (also updates organism-manifest.json prop_overrides)
     icon-manifest.json      ← icon-mapper (also updates organism-manifest.json prop_overrides)
     image-manifest.json     ← image-mapper (also updates organism-manifest.json)
     icon-library.json       ← icon-library-builder (icon name → Figma nodeId)
     image-library.json      ← image-library-builder (image name → Figma nodeId, placeholder frames)
            │
            ▼
     figma-scripts/scene.json  ← figma-instruction-writer
     figma-scripts/index.json     (via manifest_to_scene → compile_scene → execute_instructions)
            │
            ▼ [figma_execute each script]
            │
            ▼
     validation-reports/  ← design-validator
            │
     [parallel after Gate 4]
     ux-acceptance-brief.json ──────────────────────┐
     organism-manifest.json ────────────────────────┤
     lo-fi-frames/gate0-evaluation.json ────────────┤
                                                    ▼
                                          ux-evaluation-report.json  ← ux-evaluator
                                                    │
     desirability-brief.json ──────────────────────┐│
     ux-acceptance-brief.json ─────────────────────┼┤
                                                    ▼│
                                          brand-compliance-report.json  ← brand-compliance-agent
                                                    ││
     user-flows.json ─────────────────────────────┐ ││
     figma-source.json ───────────────────────────┤ ││
                                                  ▼ ▼▼
                                         prototype-links.json  ← prototype-linker
                                         [Figma prototype ready]
                                                   │
                                                   ▼
                                          delivery-package/  ← delivery-sequencer
```

---

## Quick Start

### Figma-First (existing file)
```
1. figma-intake-agent     → figma-source.json + requirements.json
2. figma-frame-extractor  → extracted-frames/
3. figma-token-extractor  → figma-tokens.json / .css / .ts
4. design-review-agent    → design-review-report.md     ← review & act on findings
5. consistency-analyser   → consistency-report.md        ← review & act on findings
6. flow-architect         → user-flows.json
7. [continue core pipeline from token-system-builder...]
8. prototype-linker       → Figma prototype links created
9. delivery-sequencer     → HANDOFF.md
```

### Brief-First (new project)
```
1.  pipeline-intake-agent    → requirements.json + ux-acceptance-brief.json (stub)
2.  requirements-analyst     → requirements.json (enriched)
3.  flow-architect           → user-flows.json
4.  wireframe-strategist     → screen-blueprints.json + ux-acceptance-brief.json (enriched)
5.  journey-mapper           → journey-map.json + ux-acceptance-brief.json (complete)
6.  lo-fi-builder            → lo-fi-frames/ + gate0-evaluation.json
                               ── GATE 0: UX/D evaluation + human review ──
7.  creative-director        → creative-direction.json + desirability-brief.json
                               ── GATE 0.5: human review ──
8.  token-system-builder     → token-map.json + Figma Variables committed
                               ── GATE 1: review token coverage ──
9.  component-architect      → component-manifest.json + component-build-plan.json
                               ── GATE 2: review build plan ──
10. component-builder        → built-component-library.json + Figma COMPONENT nodes
                               ── GATE 3: verify first-screen screenshot ──
11. organism-composer        → organism-manifest.json
12. copy-writer              → copy-manifest.json + organism-manifest.json (copy values)
13. icon-mapper              → icon-manifest.json + organism-manifest.json (icon names)
14. image-mapper             → image-manifest.json + organism-manifest.json (image slots)
15. icon-library-builder     → icon-library.json (Figma icon component nodeIds)
16. image-library-builder    → image-library.json (Figma placeholder frame nodeIds)
17. figma-instruction-writer → figma-scripts/
18. [figma_execute each script]
19. design-validator         → validation-reports/
                               ── GATE 4: parity score ≥ 0.95 ──
16. [parallel]
    ├── ux-evaluator         → ux-evaluation-report.json
    └── brand-compliance-agent → brand-compliance-report.json
17. delivery-sequencer       → HANDOFF.md
```

---

## Improvement Backlog

Opportunities identified through analysis of the pipeline's actual Figma manipulation capabilities, design system concepts, and AI design research. Items are grouped by tier and annotated with dependencies — implement items with prerequisites only after those prerequisites are in place.

---

### Priority: Now — Quick wins, no prerequisites

| # | Opportunity | Agent(s) | Notes |
|---|---|---|---|
| 1 | **Fix prototype seed spec** — update `seed-prototype-linking.yaml` to use `actions: [...]` array format instead of deprecated `action: {}` object | `seed-prototype-linking.yaml` | Bug fix; will fail silently on next run without this |
| 2 | **Screenshot input to `design-review-agent`** — pass rendered frame images to the agent alongside layout tree JSON | `design-review-agent` | Optical balance, crowded space, and perceived contrast only visible in the image; no structural changes required |
| 3 | **Validator ground truth via `figma_execute`** — read actual `node.paddingLeft`, `node.fills` etc. programmatically instead of estimating from screenshots | `design-validator` | Must come before #4 — accurate measurements make the vision pass more targeted; agents may make Figma MCP read calls even though they cannot roam the filesystem |

---

### Priority: Next — Structural correctness

| # | Opportunity | Agent(s) | Prerequisites | Notes |
|---|---|---|---|---|
| 4 | **Vision pass in `design-validator`** — send screenshot to Claude vision alongside structured checks for quality issues (hierarchy, whitespace rhythm, brand feel) | `design-validator` | #3 first | Correctness and quality are different axes; accurate measurements (#3) make vision feedback more targeted |
| 5 | **Design system binding in `figma-instruction-writer`** — implement both Figma Variable binding (`setBoundVariableForNode()`) and named styles application (`figma_get_styles` references) together in one upgrade | `figma-instruction-writer` | — | These are the same class of fix: bind to the design system instead of hardcode values; always implement together — doing one without the other is a half-measure |
| 6 | **Live component key discovery** — `component-architect` calls `figma_search_components` + `figma_get_component_details` before writing the manifest so `component_key` and `figma_node_id` are always real values | `component-architect` | — | Prerequisite for #10; without real keys, #10 builds components but can't instantiate them from published libraries |
| 7 | **Advanced Auto Layout coverage** — add `layoutWrap`, `layoutPositioning='ABSOLUTE'`, `minWidth`/`maxWidth`, `strokesIncludedInLayout` to instruction-writer rules | `figma-instruction-writer` | — | Independent of #5; wrapping grids, floating badges, and responsive constraints produce incorrect output without this |
| 8 | **Pipeline state versioning** — `delivery-sequencer` emits `pipeline-state.json` with scores, timestamps, artifact hashes, and frame property snapshots per run | `delivery-sequencer` | — | Prerequisite for #11 (diff scripts need previous state) and for DELTA mode in #17 (intake agent needs run history) |

---

### Priority: Later — New capabilities

| # | Opportunity | Agent(s) | Prerequisites | Notes |
|---|---|---|---|---|
| 9 | **Consistency auto-fix** — `consistency-analyser` generates targeted `figma_execute` snippets to correct top-N off-system values directly; implemented as a separate `consistency-fixer` delegate agent to keep `consistency-analyser` itself read-only | new: `consistency-fixer`; `consistency-analyser` delegates | — | Separating fixer from analyser resolves the conflict with the write permission list (#21): `consistency-analyser` stays read-only, `consistency-fixer` is the write-capable delegate |
| 10 | **Build components, not frames** — `figma-instruction-writer` creates proper Figma `COMPONENT` nodes with exposed properties and component sets for variant states | `figma-instruction-writer`, `organism-composer` | #6 (real component keys) | Without #6, components are built with null keys and can't be published or instantiated from libraries |
| 11 | **Diff scripts instead of rebuild scripts** — inspect current Figma frame state before generating scripts; emit add/modify/remove diffs rather than full rebuilds | `figma-instruction-writer` | #8 (pipeline state snapshots provide previous state baseline) | Diff scripts require knowing what the previous state was; #8 snapshots supply that |
| 12 | **Organism-first build order** — build and validate organisms in isolation before assembling screens from them | `organism-composer`, `figma-instruction-writer` | #10 (organisms are components; #10 must be in place) | Requires #10 because organisms are meaningless as frames — they need to be real components to be instantiated into screens |
| 13 | **Brand-compliance agent** — scores screens for brand personality alignment using a vision model + brand guidelines | new: `brand-compliance-agent` | #2 (screenshots flowing into agents) | Orthogonal to token-correctness checks; answers "does this feel like the brand", not "does this use the right hex" |
| 14 | **Motion/animation spec agent** — defines micro-interaction specs (easing curves, stagger patterns, skeleton shimmer) as part of handoff | new: `motion-spec-agent` | — | Independent; adds a new handoff artifact without touching existing agents |
| 15 | **Competitive reference ingestion** — `requirements-analyst` accepts inspiration URLs; extracts design patterns via vision model | `requirements-analyst` | — | Independent; controlled by `competitive_refs` in `pipeline.config.json` |

---

### Priority: Foundational — Infrastructure (implement as a set)

Items 16–21 form a coherent infrastructure layer. They interact closely and should be designed together even if implemented incrementally. The correct implementation order within this group is: 16 → 17 → 18 → 19 → 20 → 21.

| # | Opportunity | Agent(s) | Prerequisites | Notes |
|---|---|---|---|---|
| 16 | **`pipeline.config.json` — single input declaration point** — all inputs declared in one file; `pipeline-intake-agent` produces it through elicitation; all other agents read from it | All agents; new: `pipeline-intake-agent` produces it | — | Foundation for all other infrastructure items; must exist before #17 can write to it and before #18–21 can enforce containment rules against it |
| 17 | **`pipeline-intake-agent` — adaptive elicitation** — detects run mode (FAST / SCAN / FULL / DELTA), asks only questions not inferrable from inputs, produces `pipeline.config.json`, manages mid-pipeline checkpoints and chunked progress output | new: `pipeline-intake-agent` | #16 (config schema must be defined) | DELTA mode depends on `run-history.json` from #21 (reflection agent); DELTA unavailable until at least one prior run has been reflected |
| 18 | **Strict working directory containment** — all agents read/write only within `working_dir` and `pipeline_memory_dir`; agents may make Figma MCP read calls without this restriction; `Glob`/`Grep` outside these zones is prohibited | All agents | #16 (config defines the dirs) | Note: Figma MCP read-only calls (not filesystem reads) are permitted from any agent regardless of containment |
| 19 | **Codebase access opt-in gate** — disabled by default; `codebase.enabled: true` required; restricted to `delivery-sequencer`; enforces `exclude_patterns` | `delivery-sequencer` | #16 (config holds the gate flag) | Highest-risk access zone; the separate `consistency-fixer` agent (#9) must also be excluded from codebase access |
| 20 | **Figma MCP write permission list** — only `figma-instruction-writer`, `prototype-linker`, `organism-composer`, and `consistency-fixer` (#9) may call write tools; all other agents restricted to read-only Figma tools | All agents | — | `consistency-fixer` (#9) is explicitly added to the write-capable list here; `consistency-analyser` remains read-only |
| 21 | **Reflection agent + memory architecture** — `reflection-agent` runs after every pipeline completion; aggregates correction patterns, Figma edit diffs, console errors, and user feedback; promotes high-confidence patterns; only agent that writes to `pipeline-memory/`; `delivery-sequencer` also writes frame snapshots to `pipeline-memory/run-snapshots/` | new: `reflection-agent`; `delivery-sequencer` (snapshot write) | #8 (pipeline state snapshots), #16 (config defines memory dir) | Merges former items 17 and 23 (memory write isolation); `reflection-agent` is the sole writer except for delivery-sequencer's snapshot writes |

#### Elicitation modes (#17 detail)

| Mode | Trigger | Max questions | Gates |
|---|---|---|---|
| **FAST** | Figma URL provided | 3 | GATE 0 (single confirm) |
| **SCAN** | Structured PRD/brief provided | Gaps only (≤ 5) | GATE 0.5 (single confirm) |
| **FULL** | Vague text description, no file | 12 across 3 phases | GATE 1 (screen inventory) → GATE 2 (token preview) → GATE 3 (flow map) |
| **DELTA** | Prior run in `pipeline-memory/` | 1 (scope of change) | GATE D — requires #21 to have run at least once |

#### Mid-pipeline checkpoints (#17 detail)

```
CHECKPOINT A — After Stage 1 (requirements.json written)
  Trigger: only if confidence LOW or screen count differs from stated scope by >20%
  Skip: if HIGH confidence and FAST mode

CHECKPOINT B — After Stage 4 (token-map.json written)  ← ALWAYS
  Last cheap decision point before Figma execution (Stages 5–7 are most expensive)
  Show: coverage %, blocking gaps, auto-generate option

CHECKPOINT C — After first screen built  ← ALWAYS
  Show: screenshot of first screen for visual direction confirm
  Catching wrong direction at screen 1 costs seconds; at screen 12 costs the full build
```

#### Chunked progress output (#17 detail)

```
✓ Screen 1/12 — Login (default) built
⚠ Screen 3/12 — Signup — validation warning (contrast on submit button)
Build complete. 11 passed, 1 warning. Proceed? [Yes / Fix warning first]
```

#### Self-learning promotion thresholds (#21 detail)

```
count ≥ 3  + confidence > 0.70  →  inject as soft context (agent can override)
count ≥ 5  + confidence > 0.85  →  inject as hard constraint (agent must follow)
count ≥ 8  + confidence > 0.90  →  promote to permanent agent rule (rewrite .md)
User explicit confirmation       →  promote immediately regardless of count
User explicit rejection          →  archive to anti-patterns.json permanently
```

#### `pipeline.config.json` schema (#16)

```json
{
  "project_name": "string",
  "run_id": "string (auto-generated if omitted)",
  "platform_targets": ["ios", "android", "web", "desktop"],
  "working_dir": "/absolute/path/to/run/dir",
  "pipeline_memory_dir": "/absolute/path/to/memory/dir",
  "source": {
    "mode": "figma-first | brief-first",
    "figma_file_url": "https://www.figma.com/design/...",
    "figma_page_filter": ["page names to include; empty = all"],
    "figma_component_library_file_key": "optional second file key for published library",
    "brief_path": "relative to working_dir",
    "brand_docs_path": "relative to working_dir (optional)",
    "competitive_refs": ["URL list — improvement #15"]
  },
  "design_system": {
    "token_file_path": "relative to working_dir (optional)",
    "component_library_path": "relative to working_dir (optional)",
    "design_system_docs_path": "relative to working_dir (optional)"
  },
  "codebase": {
    "enabled": false,
    "code_components_dir": "absolute path or null",
    "storybook_stories_glob": "glob pattern within code_components_dir or null",
    "import_path_prefix": "string or null",
    "exclude_patterns": ["*.env", "*.secret", "config/**", "credentials/**", "migrations/**"]
  },
  "pipeline_memory": {
    "enabled": true,
    "read_preferences": true,
    "write_corrections": true
  }
}
```

#### Dependency graph (full backlog)

```
#1  (fix seed spec)        — no deps
#2  (screenshot → review)  — no deps
#3  (validator ground truth) → enables #4

#4  (vision pass)          — needs #3
#5  (design system binding) — no deps; variable + named styles always together
#6  (component key discovery) → enables #10
#7  (advanced auto layout) — no deps
#8  (state versioning)     → enables #11, #21 (DELTA mode)

#9  (consistency-fixer)    — no deps; separate agent preserves #20 write list
#10 (build components)     — needs #6; → enables #12
#11 (diff scripts)         — needs #8
#12 (organism-first order) — needs #10
#13 (brand compliance)     — needs #2
#14 (motion spec)          — no deps
#15 (competitive refs)     — no deps

#16 (pipeline.config)      → enables #17, #18, #19
#17 (intake agent)         — needs #16; DELTA mode needs #21
#18 (containment)          — needs #16
#19 (codebase gate)        — needs #16
#20 (Figma write list)     — no deps (documents rules; #9 must be listed)
#21 (reflection + memory)  — needs #8, #16
```
