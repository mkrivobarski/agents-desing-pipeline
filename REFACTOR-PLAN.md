# Design Pipeline Refactoring Plan
## From Ad-Hoc Shape Building → Programmatic Component-First Design

> This plan transforms the pipeline's Figma execution approach from a
> shape-drawing mindset to a component-system mindset, borrowing principles
> directly from front-end engineering.

---

## The Core Problem

The current pipeline builds screens by **drawing shapes and stamping tokens onto them**. This is how a human designer works at 2am in a hurry — not how a scalable, AI-friendly system should work.

**What the current pipeline does:**
- `figma-instruction-writer` constructs screens by creating Frames, setting fills from token values, appending children
- Components are resolved via `component-resolver` but often degrade to raw frame construction when keys are missing
- Design tokens are resolved to primitive values (hex, px) and embedded inline
- Organisms are assembled *after* screens are built — a retrospective composition pass
- The pipeline has no canonical library of components with defined API contracts

**What front-end engineering taught us:**
- Build atoms first; compose them into molecules; compose molecules into organisms
- Every component has a clear contract: inputs (props/variants), outputs (rendered UI), and states
- Components are instantiated — never drawn from scratch at the screen level
- Design tokens are bound to variables, not resolved to primitives
- Screens are assembled from component instances, not constructed node by node

---

## Refactoring Strategy: Four Structural Shifts

### Shift 1: Component Library as First-Class Citizen

The pipeline currently treats the component library as a lookup table used during manifest generation. It should be treated as a **deployed system** — the single source of truth from which every screen is assembled.

**New principle:** Nothing is drawn at screen-build time. Every visible element is either:
1. An instance of a published Figma component, or
2. An instance of a custom component built and registered in the **current run's component library**

**Impact on agents:**
- `component-resolver` becomes the **Component Architect** — it doesn't just match slots to existing components, it also *defines new components* before any screen is built
- Custom components are built as proper Figma `COMPONENT` nodes with exposed properties, variant sets, and programmatic states, before any screen receives them
- `figma-instruction-writer` **never builds from primitives** — it only instantiates

---

### Shift 2: Atomic Build Order (atoms → molecules → organisms → screens)

Currently the pipeline builds screens, then retrospectively identifies organisms. This is backward.

**New build order:**

```
Stage 0: Token System     → Figma Variables (bound, not resolved)
Stage 1: Atoms            → Button, Input, Icon, Avatar, Badge, Chip, Toggle, ...
Stage 2: Molecules        → SearchBar (Input + Icon), FormField (Label + Input + Error), ...
Stage 3: Organisms        → Header (NavBar + SearchBar), Card (Image + Heading + Body + CTA), ...
Stage 4: Screen Templates → Shell (Header + ContentArea + Footer) with slot placeholders
Stage 5: Screens          → Template instances populated with organism instances
```

Every stage only uses outputs from the stage before it. No stage reaches past its boundary.

---

### Shift 3: Components Have Programmatic States, Not Just Variant Props

Currently variants map to visual states (e.g., `state: "default"`) but these are static labels. The pipeline should model states as a **state machine** with defined transitions:

```
idle → hover → pressed → disabled
idle → loading → success | error
```

Each component definition must specify:
- **States**: `default`, `hover`, `pressed`, `focused`, `disabled`, `loading`, `success`, `error`
- **Which states are interactive** (hover, pressed) vs. data-driven (loading, error)
- **Which props change per state** (e.g., opacity 0.38 in disabled, spinner visible in loading)

Figma component sets implement this via `State` variant property. The pipeline must enforce that *every interactive component* has at least: default, hover, pressed, disabled. Every data-loading component adds: loading, error, empty.

---

### Shift 4: Pipeline Produces Intermediate Design Artifacts

Currently the pipeline only produces a final hi-fi screen. The user asked for:
**Journey maps → Lo-fi wireframes → Design tokens → Components → Screens**

Each intermediate artifact both validates the previous stage's reasoning AND grounds the next stage's execution.

```
Brief
  │
  ▼
journey-map.md          ← User journeys as narrative + mermaid flowchart
  │                        (validates requirements understanding)
  ▼
lo-fi-wireframes/        ← Low-fidelity structural wireframes in Figma
  │                        (grey boxes, real layout, real zones, no color)
  ▼
design-tokens.json       ← Token system committed to Figma Variables before
  │                        any component is built
  ▼
component-library/       ← Atoms → Molecules → Organisms built as real
  │                        Figma components with full state sets
  ▼
screen-templates/        ← Shell layouts instantiated from organisms
  │
  ▼
hi-fi-screens/           ← Final screens assembled from component instances
```

---

## Concrete Agent Changes

### New Agent: `journey-mapper`

**Purpose:** Produce a UX journey map as the very first pipeline artifact.

**Input:** `requirements.json`

**Output:** `journey-map.md` + `journey-map.json`

**What it produces:**
- Named user personas (derived from requirements)
- User journeys per persona: goals → steps → touchpoints → pain points → emotions
- A Mermaid flowchart of each journey
- A Figma representation: a dedicated "Journey Maps" page with a journey map frame per persona (sticky-note style, using rectangles and text — this is a thinking tool, not a UI)

**Why it matters:** Forces the pipeline to articulate *who is using this and why* before any screen is drawn. Catches misaligned requirements before expensive work.

---

### New Agent: `lo-fi-builder`

**Purpose:** Build structural lo-fi wireframes in Figma before any visual design.

**Input:** `screen-blueprints.json`, `user-flows.json`

**Output:** `lo-fi-frames/` directory of Figma frame node IDs + screenshot thumbnails

**Approach:**
- Creates a dedicated Figma page: "Lo-Fi Wireframes"
- Builds one frame per screen per state, using a strict greyscale palette:
  - Background: `#FFFFFF`
  - Primary surface: `#F5F5F5`
  - Secondary surface: `#E0E0E0`
  - Placeholder text: `#9E9E9E`
  - Borders: `#BDBDBD`
- Zones become named frames with auto layout
- Slots become labelled placeholder rectangles (annotated with slot_id and slot_type)
- Navigation flow arrows drawn using Figma connector lines
- No component instantiation — pure structural representation

**Why before components:** Lo-fi wireframes validate layout, hierarchy, and flow before any investment in component building. Structural problems caught at lo-fi cost nothing to fix.

---

### Refactored Agent: `token-mapper` → `token-system-builder`

**Current behaviour:** Maps tokens to blueprint slots, outputs `token-map.json` with resolved hex/px values.

**New behaviour:**
1. Reads existing design tokens from codebase/brief
2. Constructs a **Figma Variable collection** in the live Figma file (not just a JSON map)
3. Organises variables into semantic collections: `Primitives` → `Semantic` → `Component`
4. Establishes all modes required (e.g., Light/Dark)
5. Only then produces `token-map.json` — but now `token-map.json` contains `variable_id` fields for every token

**Token collection structure to create in Figma:**
```
Primitives/
  Color/Blue/100, /200, ... /900
  Color/Neutral/...
  Spacing/4, /8, /12, /16, /24, /32, /48, /64
  Radius/sm, /md, /lg, /full
  Typography/FontSize/xs, /sm, /md, /lg, /xl
  Typography/LineHeight/tight, /normal, /relaxed

Semantic/
  Color/Background/default, /elevated, /overlay
  Color/Text/primary, /secondary, /disabled, /inverse
  Color/Border/default, /subtle, /strong
  Color/Interactive/default, /hover, /pressed, /disabled
  Color/Feedback/error, /warning, /success, /info
  Spacing/Component/sm, /md, /lg
  Spacing/Layout/page-margin, /section-gap, /content-gap

Component/
  Button/Background/default, /hover, /pressed, /disabled
  Button/Text/default, /disabled
  Input/Border/default, /focused, /error, /disabled
  ...
```

**Key constraint:** `figma_batch_create_variables` and `figma_batch_update_variables` are used — never single-variable calls.

**Output adds:** Every token in `token-map.json` includes `variable_id` (Figma VariableID string). The `coverage_score` threshold increases to 90% (was 80%).

---

### Refactored Agent: `component-resolver` → `component-architect`

**Current behaviour:** Matches slots to existing library components, flags CUSTOM_REQUIRED.

**New behaviour:**

**Phase 1 — Inventory**
- Live-discover all published components (existing behaviour, kept)
- Classify each by atomic design tier: `atom`, `molecule`, `organism`

**Phase 2 — Gap analysis**
- For every slot type in all blueprints, check whether an appropriate component exists
- For each gap: define the component's API contract before it is built:
  - **Name** (PascalCase)
  - **Category** (atom/molecule/organism)
  - **Props** (typed: string, boolean, enum, number)
  - **Variants** (as a state machine: states + transitions)
  - **Slots** (named children for composable components)
  - **Token bindings** (which semantic tokens apply to which visual properties)
  - **Auto layout spec** (layoutMode, sizing behaviour, gap, padding)

**Phase 3 — Component build order**
- Topologically sort required custom components (atoms before molecules that use them)
- Produce a `component-build-plan.json` listing build order with dependencies

**Output adds:** `component-build-plan.json` in addition to `component-manifest.json`

---

### New Agent: `component-builder`

**Purpose:** Build the atoms, molecules, and organisms as real Figma COMPONENT nodes before any screen is touched.

**Pipeline position:** Runs AFTER `component-architect`, BEFORE `organism-composer`.

**Input:** `component-build-plan.json`, `token-map.json` (with `variable_id` fields)

**Output:** `built-component-library.json` (maps component names to Figma node IDs; see key stability note below)

#### Figma API contract for building component sets

The Figma Plugin API does **not** expose a `createComponentSet()` global. The correct pattern is:

1. Create N individual `COMPONENT` nodes (one per state variant)
2. Name each with its variant properties using the `Property=Value` convention: `State=Default`, `State=Hover`, etc.
3. Call `figma.combineAsVariants([comp1, comp2, ...])` — this destroys the original component nodes and returns a single new `COMPONENT_SET` node
4. Immediately read back `.id` and `.key` from the returned `COMPONENT_SET` and from each of its `.children` (the variant components) — these IDs are new and differ from the originals

```javascript
// Step 1: create individual variant components
const defaultComp = figma.createComponent();
defaultComp.name = "State=Default";
defaultComp.resize(120, 40);
// ... apply fills, auto layout, variable bindings ...

const hoverComp = figma.createComponent();
hoverComp.name = "State=Hover";
hoverComp.resize(120, 40);
// ... apply different variable bindings for hover state ...

// Step 2: place all variants in the section before combining
atomsSection.appendChild(defaultComp);
atomsSection.appendChild(hoverComp);

// Step 3: combine — this replaces the individual nodes with a COMPONENT_SET
const buttonSet = figma.combineAsVariants([defaultComp, hoverComp], atomsSection);
buttonSet.name = "Button"; // name the set

// Step 4: read back IDs immediately — the originals are gone
const setId  = buttonSet.id;
const setKey = buttonSet.key;
const variantIds = {};
for (const child of buttonSet.children) {
  variantIds[child.name] = { node_id: child.id, key: child.key };
}
```

Scripts must call `figma_execute` in one call per component set (not one call per variant), and must return the `setId`, `setKey`, and `variantIds` in the result object so that `built-component-library.json` is populated with post-combine IDs.

#### Approach per component

1. Create a dedicated Figma page: "Component Library"
2. Create a Section per atomic tier: "Atoms", "Molecules", "Organisms"
3. For each component in build order (atoms first, then molecules, then organisms):
   a. Create one `COMPONENT` node per state variant; name each `Property=Value`
   b. Apply auto layout from the component spec
   c. Bind all visual properties to Figma Variables via `setBoundVariableForNode` (never hardcoded values)
   d. Call `figma.combineAsVariants(variants, section)` to produce the `COMPONENT_SET`
   e. Read back IDs from the returned set and its children (step 4 above)
   f. Add component properties (`TEXT`, `BOOLEAN`, `INSTANCE_SWAP`) after combining — these are set on the `COMPONENT_SET` node, not on individual variants
   g. Set `componentSet.description` from the API contract
   h. For molecules/organisms: find atomic children by node ID from `built-component-library.json` using `figma.getNodeByIdAsync(nodeId)`, then call `.createInstance()` on the component node (not the set)

**State variants required for interactive components:**

```
COMPONENT_SET: Button
  State=Default   ← fills bound to Component/Button/Background/default variable
  State=Hover     ← fills bound to Component/Button/Background/hover variable
  State=Pressed   ← fills bound to Component/Button/Background/pressed variable
  State=Focused   ← fills bound to Component/Button/Background/default + stroke bound to focus ring variable
  State=Disabled  ← fills bound to Component/Button/Background/disabled variable; opacity 0.38
  State=Loading   ← same as Default + spinner child visible (BOOLEAN prop)
```

**Variable binding per variant:**
```javascript
const bgVar = figma.variables.getVariableById(tokenMap["Component/Button/Background/hover"].variable_id);
if (bgVar) {
  figma.variables.setBoundVariableForNode(hoverComp, 'fills', bgVar);
} else {
  hoverComp.fills = [{ type: 'SOLID', color: hexToRgb(tokenMap["Component/Button/Background/hover"].resolved_value) }];
}
```

#### Key stability and instantiation rules

**Component keys in Figma are only stable for published library components.** Locally created components have keys that are valid within the current file session but **cannot be used with `figma.importComponentByKeyAsync()`** (that API requires a published library component from a connected library file).

**Within the same file**, all instantiation must use node IDs, not keys:

```javascript
// CORRECT — works for locally built components in the same file
const compNode = await figma.getNodeByIdAsync(nodeId); // nodeId from built-component-library.json
if (compNode && compNode.type === 'COMPONENT') {
  const instance = compNode.createInstance();
}

// WRONG for local components — will throw "component not found in any connected library"
const comp = await figma.importComponentByKeyAsync(key);
```

`component_set_key` in `built-component-library.json` is stored for **future use only** — if this library is later published as a shared file and connected to other Figma files. It must not be used for instantiation within the current pipeline run.

**Output format:**
```json
{
  "page_id": "...",
  "page_name": "Component Library",
  "sections": {
    "atoms": "section_node_id",
    "molecules": "section_node_id",
    "organisms": "section_node_id"
  },
  "components": {
    "Button": {
      "tier": "atom",
      "component_set_id": "...",
      "component_set_key": "... (stored for future library publishing only)",
      "instantiation_method": "getNodeByIdAsync",
      "variants": {
        "State=Default": { "node_id": "...", "key": "..." },
        "State=Hover":   { "node_id": "...", "key": "..." },
        "State=Pressed": { "node_id": "...", "key": "..." },
        "State=Focused": { "node_id": "...", "key": "..." },
        "State=Disabled":{ "node_id": "...", "key": "..." }
      }
    }
  }
}
```

---

### Refactored Agent: `figma-instruction-writer` (constrained)

**Principle change:** This agent is **not allowed to create shapes** in screen frames. Its only permitted operations in screen-level scripts are:

1. Instantiate a locally-built component via `figma.getNodeByIdAsync(nodeId)` → `.createInstance()` — using node IDs from `built-component-library.json`
2. Instantiate a published library component via `figma.importComponentByKeyAsync(key)` — **only** for components that exist in an externally connected library (i.e. have `source: "external_library"` in `built-component-library.json`)
3. `instance.setProperties()` — set variant selection and exposed props
4. `figma.variables.setBoundVariableForNode()` — bind token overrides on instances where per-instance variable overrides are needed
5. Frame/Section creation (screen shell only — one outer frame per screen)
6. Auto layout configuration (on the shell frame only)

**Instantiation method selection:**
```javascript
// For components built in this run (built-component-library.json source: "local")
const compNode = await figma.getNodeByIdAsync(entry.variants["State=Default"].node_id);
if (!compNode || compNode.type !== 'COMPONENT') {
  // ESCALATION — do not fall back to a placeholder shape
  return { screen_id, escalation: `Component node ${entry.variants["State=Default"].node_id} not found` };
}
const instance = compNode.createInstance();

// For published library components (source: "external_library") only
const comp = await figma.importComponentByKeyAsync(entry.component_set_key);
const instance = comp.createInstance();
```

It explicitly **cannot** call in a screen context:
- `figma.createRectangle()`
- `figma.createText()`
- `figma.createEllipse()`
- `figma.createFrame()` (except the single outermost screen shell)

Any element that requires a primitive shape must exist inside a component built by `component-builder`.

**What changes in practice:**
- The script template replaces the `importComponentByKeyAsync` default with `getNodeByIdAsync` for all locally built components
- The "placeholder rectangle" fallback in Rule 8 is removed — missing component nodes escalate immediately, forcing `component-builder` to be rerun rather than silently shipping broken screens
- The validation layer rejects scripts that contain `createRectangle`, `createText`, etc. outside of a `COMPONENT` construction context

---

### Refactored Agent: `organism-composer` (placement manifest only)

Organisms are now **built** in `component-builder` as proper Figma `COMPONENT` nodes. `organism-composer` runs after `component-builder` and before `figma-instruction-writer`. Its sole job is to produce the placement manifest that tells `figma-instruction-writer` which organism to place in which zone of which screen.

**Why `organism-composer` cannot merge with `figma-instruction-writer`:** `figma-instruction-writer` scripts are screen-scoped and executed one per screen. `organism-composer` has cross-screen visibility — it identifies shared organisms and can specify per-screen overrides in one pass. Keeping them separate preserves this.

**Sequencing fix:** `organism-composer` does not need screen frame node IDs. It produces a placement spec (`organism-manifest.json`) that maps `{organism_id, screen_id, zone_id, variant_overrides}`. `figma-instruction-writer` reads this manifest, creates the screen shell frame (which generates a new node ID at runtime), and then calls `getNodeByIdAsync` on the organism component and instantiates it into the shell.

`organism-manifest.json` entries do **not** contain `parentId` — that field is resolved at script-execution time by `figma-instruction-writer` using the screen shell frame it just created.

**`organism-composer` responsibilities:**
1. Read `built-component-library.json` — verify every organism component referenced in blueprints was built
2. Read `screen-blueprints.json` — map organism types to screen zones
3. Identify shared organisms (same organism used in multiple screens)
4. Produce `organism-manifest.json` — one entry per `{organism_id, screen_id, zone_id}` tuple, with per-screen prop/variant overrides but **no** `parentId`
5. Flag any zone that needs an organism not present in `built-component-library.json` — this is a `component-builder` gap, not an `organism-composer` failure

---

## Updated Pipeline Flow

```
╔══════════════════════════════════════════════════════╗
║  BRIEF-FIRST PATH                                     ║
╠══════════════════════════════════════════════════════╣
║                                                       ║
║  [Brief / PRD / Brand docs]                           ║
║       │                                               ║
║       ▼                                               ║
║  requirements-analyst → requirements.json             ║
║       │                                               ║
║       ▼                                               ║
║  journey-mapper       → journey-map.md                ║  ← NEW: intermediate artifact
║       │               → journey-map.json              ║
║       │               → Figma: Journey Maps page      ║
║       ▼                                               ║
║  flow-architect       → user-flows.json               ║
║       │                                               ║
║       ▼                                               ║
║  wireframe-strategist → screen-blueprints.json        ║
║       │                                               ║
║       ▼                                               ║
║  lo-fi-builder        → lo-fi-frames/ (screenshots)   ║  ← NEW: intermediate artifact
║       │               → Figma: Lo-Fi Wireframes page  ║
║       │                                               ║
╠═══════╪═══════════════════════════════════════════════╣
║       │  DESIGN SYSTEM PHASE                          ║
╠═══════╪═══════════════════════════════════════════════╣
║       ▼                                               ║
║  token-system-builder → token-map.json (with var IDs) ║  ← REFACTORED: builds Figma vars
║       │               → Figma: Variables committed    ║
║       ▼                                               ║
║  component-architect  → component-manifest.json       ║  ← REFACTORED: API contracts
║       │               → component-build-plan.json     ║
║       ▼                                               ║
║  component-builder    → built-component-library.json  ║  ← NEW: builds atoms/molecules/organisms
║       │               → Figma: Component Library page ║
║       │                                               ║
╠═══════╪═══════════════════════════════════════════════╣
║       │  SCREEN ASSEMBLY PHASE                        ║
╠═══════╪═══════════════════════════════════════════════╣
║       ▼                                               ║
║  organism-composer    → organism-manifest.json        ║  ← REFACTORED: orchestrates, not builds
║       │                                               ║
║       ▼                                               ║
║  figma-instruction-writer → figma-scripts/            ║  ← CONSTRAINED: instances only
║       │                                               ║
║       ▼ [figma_execute each script]                   ║
║       │                                               ║
║       ▼                                               ║
║  design-validator     → validation-reports/           ║
║       │                                               ║
║       ▼                                               ║
║  prototype-linker     → Figma prototype reactions     ║
║       │                                               ║
║       ▼                                               ║
║  delivery-sequencer   → HANDOFF.md                    ║
╚══════════════════════════════════════════════════════╝
```

---

## Intermediate Artifacts Produced Along the Way

| Stage | Artifact | Purpose |
|---|---|---|
| `journey-mapper` | `journey-map.md` | Human-readable narrative of user goals and pain points |
| `journey-mapper` | Figma Journey Maps page | Visual journey map — frames per persona, sticky-note layout |
| `lo-fi-builder` | `lo-fi-frames/*.png` | Screenshots of structural wireframes for review gate |
| `lo-fi-builder` | Figma Lo-Fi Wireframes page | Grey-box layout with zone annotations |
| `token-system-builder` | `token-map.json` (with var IDs) | Token registry with live Figma Variable IDs |
| `token-system-builder` | Figma Variables (committed) | Token system bound in Figma before any component is built |
| `component-architect` | `component-build-plan.json` | Topologically sorted component specs with API contracts |
| `component-builder` | `built-component-library.json` | Registry of all built component node IDs and keys |
| `component-builder` | Figma Component Library page | Full atomic design library: atoms → molecules → organisms |

---

## Checkpoint Gates (When to Stop and Review)

```
GATE 0 — After journey-map.md
  Review: Do the user journeys reflect the actual brief?
  Cost to fix: free (no Figma work done yet)

GATE 1 — After lo-fi wireframes
  Review: Does the structure, layout, and flow feel right?
  Cost to fix: cheap (lo-fi adjustments only, no tokens/components)

GATE 2 — After token system committed to Figma
  Review: Is the token hierarchy correct? Are modes (Light/Dark) set up?
  Cost to fix: moderate (variables can be batch-updated, but component
  bindings will need recheck if semantics change)

GATE 3 — After component library built (first atom pass)
  Review: Screenshot of Component Library page. Do the atoms look right?
  Cost to fix: moderate (component rebuild, but screens not yet touched)

GATE 4 — After first screen assembled (existing CHECKPOINT C)
  Review: Screenshot of screen 1. Is the direction correct?
  Cost to fix: expensive if architecture is wrong; cheap if props/variants
```

---

## Files to Create / Modify

### New agent files
- `agents/design-pipeline/journey-mapper.md`
- `agents/design-pipeline/lo-fi-builder.md`
- `agents/design-pipeline/component-builder.md`

### Modified agent files
- `agents/design-pipeline/token-mapper.md` → `token-system-builder.md`
- `agents/design-pipeline/component-resolver.md` → `component-architect.md`
- `agents/design-pipeline/organism-composer.md` — placement-manifest-only role; remove Figma write calls
- `agents/design-pipeline/figma-instruction-writer.md` — replace `importComponentByKeyAsync` default with `getNodeByIdAsync`; remove shape-creation fallbacks
- `agents/design-pipeline/design-validator.md` — increase extraction depth to 6; add instance validation category; update ground-truth snippet

### New seed files
- `agents/design-pipeline/seeds/seed-journey-map.yaml`
- `agents/design-pipeline/seeds/seed-lo-fi-wireframes.yaml`
- `agents/design-pipeline/seeds/seed-token-system.yaml` (replaces `seed-token-mapping.yaml`)
- `agents/design-pipeline/seeds/seed-component-build-plan.yaml` (new from architect)
- `agents/design-pipeline/seeds/seed-component-library.yaml` (new builder)

### Updated docs
- `agents/design-pipeline/README.md` — updated pipeline map and agent table
- `agents/design-pipeline/seeds/pipeline-runner.md` — updated stage list

---

## Implementation Order

Implement in this sequence (each batch unblocks the next):

**Batch 1 — Pre-design artifacts (no Figma deps)**
1. `journey-mapper.md` + seed
2. `lo-fi-builder.md` + seed

**Batch 2 — Design system foundation**
3. `token-system-builder.md` + seed (refactor of token-mapper)
4. `component-architect.md` + seed (refactor of component-resolver)

**Batch 3 — Component construction**
5. `component-builder.md` + seed (new; includes `combineAsVariants` contract and key-stability rules)

**Batch 4 — Screen assembly**
6. Update `organism-composer.md` (placement-manifest role; remove Figma write calls)
7. Update `figma-instruction-writer.md` (`getNodeByIdAsync` instantiation; remove shape fallbacks)

**Batch 5 — Validation**
8. Update `design-validator.md` (depth-6 extraction; instance validation category)

**Batch 6 — Docs and runner update**
9. Update `README.md`
10. Update `seeds/pipeline-runner.md`

---

## What This Does NOT Change

- `requirements-analyst` — unchanged; still produces `requirements.json`
- `flow-architect` — unchanged; still produces `user-flows.json`
- `wireframe-strategist` — unchanged; still produces `screen-blueprints.json`
- `design-review-agent` — unchanged; still runs parallel review track
- `consistency-analyser` — unchanged
- `prototype-linker` — unchanged
- `delivery-sequencer` — unchanged

## What Requires Non-Trivial Updates Beyond the Four Shifts

### `design-validator` — instance-aware validation required

Currently marked as "unchanged" but **must be updated**. The ground-truth extraction script traverses 3 levels deep (`extractNode(screen, 3)`). A screen built from component instances is structured as:

```
Screen Frame (level 0)
  └─ Organism instance (level 1)
       └─ Molecule instance (level 2)
            └─ Atom instance (level 3)
                 └─ Fill / text / shape (level 4+)
```

At level 3, fills are inside the atom component — not on the atom instance node itself. The current extractor reads level-3 nodes and finds no fills, causing every color check to report FAIL.

**Required changes to `design-validator`:**

1. **Increase extraction depth** from 3 to 6 levels in the ground-truth script to reach actual rendered nodes

2. **Add instance validation category** alongside the existing 7 categories:
   - Check that the instantiated component node ID matches the expected entry in `built-component-library.json`
   - Check that `instance.variantProperties` matches the expected variant from `organism-manifest.json`
   - Check that exposed component properties (`instance.componentProperties`) match the props spec

3. **Follow instance overrides** when reading fill values: after `extractNode`, for any node of type `INSTANCE`, also read `node.overrides` to capture per-instance fill/text overrides applied by `figma-instruction-writer`

4. **Updated ground-truth snippet:**
```javascript
function extractNode(node, depth) {
  const base = { id: node.id, name: node.name, type: node.type,
                 width: node.width, height: node.height };
  // ... existing property extraction ...

  // NEW: capture instance-specific data
  if (node.type === 'INSTANCE') {
    base.mainComponent = node.mainComponent ? { id: node.mainComponent.id } : null;
    base.variantProperties = node.variantProperties || null;
    base.componentProperties = node.componentProperties || null;
    base.overrides = node.overrides ? node.overrides.map(o => ({
      id: o.id, overriddenFields: o.overriddenFields
    })) : [];
  }

  if (depth > 0 && 'children' in node) {
    base.children = node.children.map(c => extractNode(c, depth - 1));
  }
  return base;
}
return extractNode(screen, 6); // depth 6, was 3
```

5. **Parity score recalibration**: the threshold remains ≥ 0.95, but the check set expands — instance-validation checks are now worth the same weight as spacing/color checks

---

## Success Criteria

A pipeline run that follows this refactoring should produce:

1. A Figma file with **five distinct pages**: Journey Maps, Lo-Fi Wireframes, Component Library, Hi-Fi Screens, Prototype
2. A component library where every interactive component has ≥ 4 states (default, hover, pressed, disabled)
3. All fills/colors in all components **variable-bound** — zero hardcoded hex values
4. All spacing/radius/typography in all components **variable-bound** — zero hardcoded px values
5. Hi-fi screens composed entirely of component instances — no raw shapes
6. Design-to-code parity score ≥ 0.9 on first `figma_check_design_parity` run (because token names match both sides)
