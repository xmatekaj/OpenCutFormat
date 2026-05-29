# OCF — Open Cut Format Specification v1.0.0

> **Status:** Draft 1.0.0  
> **Date:** 2026-05-26  
> **Related documents:** [geometry.md](geometry.md) · [operations.md](operations.md) · [schema.json](schema.json)

---

## 1. Introduction

### 1.1 Purpose

OCF (Open Cut Format) is an open, machine-readable file format for sheet-metal and flat-material cutting projects. It encodes the complete lifecycle of a cutting job:

1. **Geometry** — part shapes as contours with SVG path data
2. **Materials** — sheet stock definitions
3. **Nesting** — placement of parts on sheets
4. **Technology** — machining operations assigned to contours
5. **Toolpaths** — (optional) pre-computed motion paths for simulation

### 1.2 Scope

OCF targets the following manufacturing processes on flat/sheet material:
- Laser cutting (CO₂, fiber, diode)
- Plasma cutting
- Oxyfuel cutting
- Waterjet cutting (pure and abrasive)
- Bevel cutting (all profiles: A, V, Y, X, K)
- CNC milling (contour and pocket)
- Drilling, tapping, countersinking
- Punching, nibbling, forming
- Laser marking, engraving, etching, scribing

OCF does **not** cover: 3D printing, multi-axis turning, injection moulding, or any process not involving flat-sheet material.

### 1.3 File Format

An OCF file is a UTF-8 encoded JSON text file. The recommended file extension is `.ocf`.

The root value of the file must be a JSON object conforming to this specification.

### 1.4 Internal Unit System

**OCF always stores all distances and coordinates in millimetres.** This is fixed and not configurable. There is no ambiguity about what any numeric value means.

The `meta.units` field serves two purposes:
1. **UI display preference** — the application may display values to the operator in inches if `"inch"` is set, even though the underlying storage is always mm.
2. **Postprocessor hint** — the postprocessor reads this field to decide which G-code unit command to emit (G21 for mm, G20 for inches) and whether to convert the toolpath coordinates.

**Consequences for machine operators using inch-based machines:**
- The OCF file contains geometry in mm.
- The postprocessor converts mm → inches when generating G-code for an inch-based controller.
- Setting `meta.units: "inch"` signals this intent to the postprocessor; it does not change any numeric value in the OCF file itself.

This design avoids the ambiguity that would arise if the same field name `thickness_mm` could mean either millimetres or inches depending on a flag elsewhere in the file.

---

## 2. Top-Level Structure

```
Root Object
├── meta          (object, required)    Project metadata
├── library       (object, required)    Part and material definitions
└── workspaces    (array,  required)    Nesting layouts and operations
```

Example skeleton:

```json
{
  "meta": { ... },
  "library": {
    "materials": [ ... ],
    "parts": [ ... ]
  },
  "workspaces": [ ... ]
}
```

Additional top-level keys are permitted; conforming parsers must ignore unknown keys.

---

## 3. `meta` Object

The `meta` object identifies the file format version and records provenance information.

| Field | Type | Required | Description |
|---|---|---|---|
| `ocf_version` | string | **YES** | Semantic version of the OCF schema (e.g. `"1.0.0"`) |
| `schema_url` | string | no | URL of the canonical JSON Schema for this version |
| `created_at` | string | no | ISO 8601 creation timestamp |
| `modified_at` | string | no | ISO 8601 last-modification timestamp |
| `application` | object | no | Software that wrote the file |
| `application.name` | string | no | Application name |
| `application.version` | string | no | Application version |
| `author` | string | no | Person or organization that created the file |
| `description` | string | no | Human-readable description of the project |
| `units` | string | no | UI display and postprocessor hint: `"mm"` (default) or `"inch"`. Storage is always mm — see §1.4. |
| `project` | object | no | Job-tracking fields |
| `project.name` | string | no | Project or job name |
| `project.customer` | string | no | Customer name |
| `project.order_number` | string | no | Order or work-order number |
| `project.part_number` | string | no | Part or drawing number |
| `project.revision` | string | no | Drawing revision |
| `project.notes` | string | no | Free-text notes |

**Example:**

```json
"meta": {
  "ocf_version": "1.0.0",
  "schema_url": "https://opencut.io/schema/1.0.0/schema.json",
  "created_at": "2026-05-26T08:30:00Z",
  "modified_at": "2026-05-26T09:15:00Z",
  "units": "mm",
  "application": {
    "name": "OmniCut",
    "version": "1.0.0"
  },
  "author": "Jan Kowalski",
  "project": {
    "name": "Bracket Assembly 2026",
    "customer": "ACME Manufacturing",
    "order_number": "PO-20260526-001",
    "part_number": "BRK-042",
    "revision": "B"
  }
}
```

---

## 4. `library` Object

The library stores reusable definitions. Geometry defined here is the canonical, untransformed source of truth. Nothing in the library carries a position on a sheet.

### 4.1 `library.materials` Array

An optional list of named materials. Workspaces can reference these by `id` instead of repeating material data.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **YES** | Unique identifier, unique within the file |
| `name` | string | **YES** | Human-readable name (e.g. `"S355 Structural Steel"`) |
| `type` | string | **YES** | Material category (see table below) |
| `density_kg_m3` | number | no | Density in kg/m³ (for mass calculations) |
| `hardness_hv` | number | no | Vickers hardness |
| `yield_strength_mpa` | number | no | Yield strength in MPa |
| `notes` | string | no | Free-text notes |

**`type` values:**

| Value | Description |
|---|---|
| `STEEL` | Carbon and structural steels (S235, S355, ...) |
| `STAINLESS_STEEL` | Stainless / inox (304, 316, ...) |
| `ALUMINUM` | Aluminium alloys |
| `COPPER` | Copper |
| `BRASS` | Brass / bronze alloys |
| `TITANIUM` | Titanium alloys |
| `NICKEL_ALLOY` | Inconel, Hastelloy, ... |
| `PLASTIC` | Acrylic, polycarbonate, HDPE, ... |
| `FOAM` | Foam materials |
| `WOOD` | Wood and wood composites |
| `STONE` | Granite, marble, ... |
| `COMPOSITE` | CFRP, fibreglass, ... |
| `OTHER` | Any other material |

**Example:**

```json
"materials": [
  {
    "id": "mat_s355_6mm",
    "name": "S355 Structural Steel",
    "type": "STEEL",
    "density_kg_m3": 7850,
    "yield_strength_mpa": 355
  },
  {
    "id": "mat_304_3mm",
    "name": "304 Stainless Steel",
    "type": "STAINLESS_STEEL",
    "density_kg_m3": 7930
  }
]
```

### 4.2 `library.parts` Array

The core of the library. Each `part` defines the geometry of one unique part shape. A part appears on a sheet as one or more `instances` in a workspace.

| Field                       | Type     | Required | Description                               |
| -----------------------------| ----------| ----------| -------------------------------------------|
| `id`                        | string   | **YES**  | Unique identifier, unique within the file |
| `name`                      | string   | no       | Human-readable name                       |
| `description`               | string   | no       | Description                               |
| `tags`                      | string[] | no       | Searchable tags                           |
| `geometry`                  | object   | **YES**  | Geometric definition (see §4.2.1)         |
| `properties`                | object   | no       | Computed geometric properties             |
| `compatible_thickness_min_mm` | number | no | Minimum sheet thickness this part is designed for (mm) |
| `compatible_thickness_max_mm` | number | no | Maximum sheet thickness this part is designed for (mm) |
| `default_operations`        | object[] | no       | Suggested default machining operations    |

#### 4.2.1 `geometry` Object

| Field | Type | Required | Description |
|---|---|---|---|
| `bounding_box` | object | no | Axis-aligned bounding box |
| `bounding_box.min` | [x,y] | no | Minimum corner |
| `bounding_box.max` | [x,y] | no | Maximum corner |
| `contours` | object[] | **YES** | List of contours (see §4.2.2) |
| `source_format` | string | no | Format the geometry was imported from: `"DXF"`, `"SVG"`, `"STEP"`, `"IGES"`, `"OCF"` |
| `source_data` | string | no | Base64-encoded original source file (lossless round-trip) |

#### 4.2.2 Contour Object

A contour is a closed or open loop encoded as a standard SVG path data string. See [geometry.md](geometry.md) for full rules and path syntax.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **YES** | Unique within the part |
| `role` | string | **YES** | `OUTER`, `INNER`, `OPEN` |
| `winding` | string | no | `CCW` (recommended for OUTER), `CW` (recommended for INNER) |
| `tags` | string[] | no | Used for operation targeting (e.g. `"hole"`, `"slot"`, `"thread-m8"`) |
| `d` | string | **YES** | SVG path data string (see [geometry.md §3](geometry.md#3-svg-path-data-syntax)) |

`role` semantics:
- `OUTER` — the external profile of the part (always one per part)
- `INNER` — a cutout or hole inside the part
- `OPEN` — a non-closed path (for scoring, marking, or partial cuts)

**Example contour:**

```json
{
  "id": "contour_outer",
  "role": "OUTER",
  "winding": "CCW",
  "tags": ["profile"],
  "d": "M 10 0 L 190 0 A 10 10 0 0 0 200 10 L 200 140 A 10 10 0 0 0 190 150 L 10 150 A 10 10 0 0 0 0 140 L 0 10 A 10 10 0 0 0 10 0 Z"
}
```

All coordinates in the `d` string are stored in millimetres (see §1.4). All contour IDs within a part must be unique.

#### 4.2.4 `properties` Object

Computed read-only values. Parsers should recompute these; the stored values are advisory.

| Field | Type | Description |
|---|---|---|
| `area_mm2` | number | Area enclosed by OUTER contour minus INNER contours |
| `perimeter_mm` | number | Total cut length (all contours) |
| `width_mm` | number | Bounding box width |
| `height_mm` | number | Bounding box height |

#### 4.2.5 `default_operations` Array

Optional suggested operations. Each entry has the same structure as workspace operations (§6.2) but references contours by `contour_id` within this part, not instance IDs.

When a workspace instantiates this part:
1. If the instance has no `override_operations`, the `default_operations` are used.
2. If the instance has `override_operations`, they replace the defaults entirely.
3. If the instance has `extend_operations`, they merge on top of the defaults.

---

## 5. `workspaces` Array

A workspace represents one physical sheet being processed. A file may contain multiple workspaces (e.g., nesting across several sheets of the same material, or sheets for different machines).

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | **YES** | Unique identifier |
| `name` | string | no | Human-readable name |
| `sheet` | object | **YES** | Sheet definition (§5.1) |
| `machine` | object | no | Target machine (§5.2) |
| `nesting_params` | object | no | Nesting parameters (§5.3) |
| `instances` | object[] | **YES** | Part instances on the sheet (§5.4) |
| `operations` | object[] | no | Workspace-level machining operations (§6) |
| `constraints` | object[] | no | Constraints between instances (§5.5) |
| `toolpaths` | object[] | no | Pre-computed toolpaths (§7) |
| `statistics` | object | no | Computed statistics (§5.6) |

### 5.1 `sheet` Object

| Field | Type | Required | Description |
|---|---|---|---|
| `material_ref` | string | no | ID of a `library.materials` entry |
| `material_inline` | object | no | Inline material definition (if not using ref) |
| `thickness_mm` | number | **YES** | Sheet thickness in mm |
| `width_mm` | number | **YES** | Sheet width (X dimension) in mm |
| `height_mm` | number | **YES** | Sheet height (Y dimension) in mm |
| `grain_direction_deg` | number | no | Grain direction in degrees from X-axis (0 = along X) |
| `is_remnant` | boolean | no | True if this is a remnant sheet (default: false) |
| `stock_id` | string | no | Warehouse/stock identifier |
| `surface_finish` | string | no | `"MILL"`, `"PICKLED"`, `"GALVANIZED"`, `"PAINTED"`, `"OTHER"` |

Both `material_ref` and `material_inline` are optional. A sheet with no material information is valid (useful for quick setups or when material tracking is managed externally).

### 5.2 `machine` Object

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | no | Machine identifier |
| `name` | string | no | Machine name |
| `type` | string or string[] | no | One or more machine categories. Use a string for single-process machines, an array for combo machines (e.g. `["PLASMA", "MILLING"]`). Valid values: `"LASER"`, `"PLASMA"`, `"WATERJET"`, `"OXYFUEL"`, `"MILLING"`, `"PUNCHING"`, `"OTHER"` |
| `manufacturer` | string | no | Manufacturer name |
| `model` | string | no | Machine model |
| `post_processor` | string | no | Post-processor identifier |
| `capabilities` | string[] | no | List of `op_type` values the machine supports |
| `bed_width_mm` | number | no | Machine bed width |
| `bed_height_mm` | number | no | Machine bed height |

**Examples:**

Single-process machine:
```json
"machine": {
  "name": "Fiber Laser 5kW",
  "type": "LASER",
  "capabilities": ["LASER_CUT", "MARKING", "ENGRAVING", "FILM_REMOVAL"]
}
```

Combo machine (plasma table with drilling head):
```json
"machine": {
  "name": "Plasma + Drill Combo",
  "type": ["PLASMA", "MILLING"],
  "capabilities": ["PLASMA_CUT", "BEVEL_CUT", "DRILLING", "TAPPING", "MILLING_CONTOUR"]
}
```

### 5.3 `nesting_params` Object

Parameters that were used (or should be used) by the nesting algorithm.

| Field | Type | Default | Description |
|---|---|---|---|
| `part_spacing_mm` | number | 5.0 | Minimum gap between parts |
| `sheet_edge_clearance_mm` | number | 10.0 | Minimum clearance from sheet edges |
| `rotation_step_deg` | number | 90 | Allowed rotation increments (0 = free rotation, 90 = 4 positions, 180 = 2 positions, 360 = fixed) |
| `allow_mirror` | boolean | false | Whether parts may be mirrored |
| `allow_common_lines` | boolean | false | Whether common-line (bridge) cutting is permitted |
| `sequence_strategy` | string | `"INSIDE_OUT"` | Cut sequence: `"INSIDE_OUT"`, `"OUTSIDE_IN"`, `"LEFT_TO_RIGHT"`, `"NEAREST_NEIGHBOUR"`, `"CUSTOM"` |
| `optimization_target` | string | `"MATERIAL_UTILIZATION"` | `"MATERIAL_UTILIZATION"`, `"CUT_TIME"`, `"TRAVEL_DISTANCE"` |
| `max_computation_s` | number | 60 | Maximum time allowed for nesting computation |

### 5.4 `instances` Array

Each entry represents one physical copy of a part placed on the sheet.

| Field | Type | Required | Description |
|---|---|---|---|
| `instance_id` | string | **YES** | Unique within the workspace |
| `part_ref` | string | **YES** | References `library.parts[*].id` |
| `transform` | object | **YES** | Position and orientation |
| `transform.x` | number | **YES** | X offset from sheet origin (mm) |
| `transform.y` | number | **YES** | Y offset from sheet origin (mm) |
| `transform.rotation_deg` | number | 0 | Rotation in degrees, counterclockwise |
| `transform.mirror_x` | boolean | false | Mirror about X axis (reflect across bounding box center) |
| `transform.mirror_y` | boolean | false | Mirror about Y axis (reflect across bounding box center) |
| `locked` | boolean | false | If true, nesting optimizer must not move this instance |
| `priority` | integer | 0 | Higher priority instances are placed first |
| `label` | string | no | Display label override |
| `override_operations` | object[] | no | Fully replaces part default_operations for this instance |
| `extend_operations` | object[] | no | Merges on top of part default_operations for this instance |

**Transform Semantics:**

Mirror operations are anchored to the **bounding box center** of the part, not the library origin. This ensures UI predictability: when a user clicks "Mirror", the part flips around its geometric center, not around (0,0).

Transform order:
1. `mirror_x` and `mirror_y` (both relative to bounding box center)
2. `rotation_deg` (rotation around bounding box center)
3. Translation by (`x`, `y`)

See [geometry.md §1.2](geometry.md#12-transform-order--mirror-reference-point) for detailed algorithmic implementation.

### 5.5 `constraints` Array

Constraints describe relationships between instances that must be preserved during nesting and toolpath computation.

#### COMMON_LINE_CUTTING

Two collinear edges from adjacent parts share a single cut pass. With SVG path encoding, edges are identified by contour ID and **command index range** within the `d` string.

```json
{
  "id": "con_001",
  "type": "COMMON_LINE_CUTTING",
  "entities": [
    {
      "instance_id": "inst_001",
      "contour_id": "outer",
      "d_command_index": 1,
      "d_command_count": 1
    },
    {
      "instance_id": "inst_002",
      "contour_id": "outer",
      "d_command_index": 3,
      "d_command_count": 1
    }
  ],
  "strategy": "SINGLE_PASS",
  "shared_segment_tolerance_mm": 0.05,
  "notes": "Bottom edge of plate A (command 1) cuts together with top edge of plate B (command 3)"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `instance_id` | string | **YES** | Reference to a workspace instance |
| `contour_id` | string | **YES** | Reference to a contour within the part |
| `d_command_index` | integer | **YES** | Zero-based index of the first SVG command in the `d` string (e.g., 0 = M, 1 = first L, etc.) |
| `d_command_count` | integer | no | Number of consecutive commands to include (default 1). Use >1 for multi-segment edges (e.g., L + A + L) |

**SVG Command Counting Rules:**
- `M` (Move) = command 0
- First `L`, `A`, `C`, `Q`, etc. = command 1
- Subsequent draw commands increment the index
- `Z` (Close) counts as a command

**Example:** For path `"M 0 0 L 150 0 L 150 100 L 0 100 Z"`:
- Index 0: `M 0 0`
- Index 1: `L 150 0` ← first edge
- Index 2: `L 150 100`
- Index 3: `L 0 100` ← third edge
- Index 4: `Z`

`strategy`: 
- `"SINGLE_PASS"` — cut once, both parts gain the edge
- `"DOUBLE_PASS"` — cut twice with offset for kerf
- `"SHARED_EDGE"` — parts touch, zero gap

**Implementation Note:** CAM systems must parse the SVG path, extract commands by index, and verify geometric collinearity (tolerance ≤ `shared_segment_tolerance_mm`).

#### FIXED_POSITION

This instance must not be moved by the nesting optimizer.

```json
{
  "type": "FIXED_POSITION",
  "instance_id": "inst_001"
}
```

#### RELATIVE_POSITION

Instance B must maintain a fixed offset from instance A.

```json
{
  "type": "RELATIVE_POSITION",
  "reference_instance_id": "inst_001",
  "target_instance_id": "inst_002",
  "offset_x": 50.0,
  "offset_y": 0.0
}
```

#### MIN_DISTANCE

Ensure a minimum distance between two instances (beyond the global `part_spacing_mm`).

```json
{
  "type": "MIN_DISTANCE",
  "instance_ids": ["inst_001", "inst_002"],
  "distance_mm": 20.0
}
```

#### SAME_ROTATION

All listed instances must have the same rotation angle.

```json
{
  "type": "SAME_ROTATION",
  "instance_ids": ["inst_003", "inst_004", "inst_005"]
}
```

### 5.6 `statistics` Object

Computed values from nesting/CAM. Stored for reference; parsers may recompute.

| Field | Type | Description |
|---|---|---|
| `utilization_pct` | number | Sheet material utilization (0–100) |
| `total_cut_length_mm` | number | Total cut path length |
| `total_rapid_length_mm` | number | Total rapid (non-cutting) travel |
| `estimated_time_s` | number | Estimated machining time in seconds |
| `part_count` | integer | Number of instances |
| `pierce_count` | integer | Total number of pierce operations |
| `computed_at` | string | ISO 8601 timestamp of computation |

---

## 6. Operations Model

### 6.1 Overview

Operations assign machining technology to contours. They follow a three-level priority chain:

```
library.parts[*].default_operations     (lowest priority, part defaults)
        ↓ overridden by
workspaces[*].operations                 (workspace/sheet defaults)
        ↓ overridden by
workspaces[*].instances[*].override_operations   (highest priority, per-instance)
```

`extend_operations` at instance level merges with (rather than replaces) defaults.

#### Per-contour technology assignment

Different contours often require different cutting parameters (speed, pierce time, quality) and different lead-in/out geometry (a small hole needs a short LINE lead-in; a large outer profile needs an ARC).

The primary mechanism for this is **multiple operations of the same `op_type`**, each targeting a different contour group via `target`. This keeps the schema flat and composable:

```json
"operations": [
  {
    "op_type": "LASER_CUT",
    "target": { "contour_role": "OUTER" },
    "params": { "speed_mm_min": 2800, "pierce_time_ms": 600, "quality": "HIGH" },
    "toolpath_hints": { "lead_in": { "type": "ARC", "radius_mm": 5.0 },
                        "lead_out": { "type": "ARC", "radius_mm": 4.0, "overcut_mm": 0.5 } }
  },
  {
    "op_type": "LASER_CUT",
    "target": { "contour_tags": ["bolt-hole"] },
    "params": { "speed_mm_min": 4500, "pierce_time_ms": 180, "quality": "HIGH" },
    "toolpath_hints": { "lead_in": { "type": "LINE", "length_mm": 1.5 },
                        "lead_out": { "type": "LINE", "length_mm": 1.0, "overcut_mm": 0.2 } }
  },
  {
    "op_type": "LASER_CUT",
    "target": { "contour_ids": ["contour_center_hole"] },
    "params": { "speed_mm_min": 3800, "pierce_time_ms": 400, "quality": "MAX" },
    "toolpath_hints": { "lead_in": { "type": "ARC", "radius_mm": 3.0 },
                        "lead_out": { "type": "ARC", "radius_mm": 2.5, "overcut_mm": 0.3 } }
  }
]
```

Two operations of the same `op_type` must not overlap in their targeted contours.

### 6.2 Operation Object

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | no | Unique identifier for cross-referencing |
| `op_type` | string | **YES** | Operation type (see [operations.md](operations.md)) |
| `target` | object | no | Which contours this operation applies to |
| `params` | object | **YES** | Operation-specific parameters |
| `toolpath_hints` | object | no | Hints for toolpath computation |
| `priority` | integer | 0 | Execution priority within a workspace (lower = executed first) |
| `enabled` | boolean | true | Set false to temporarily disable without removing |
| `notes` | string | no | Operator notes |

### 6.3 `target` Object

Controls which contours receive this operation.

| Field | Type | Default | Description |
|---|---|---|---|
| `scope` | string / string[] | `"ALL"` | `"ALL"` or a list of `instance_id` values |
| `contour_role` | string | `"ALL"` | `"OUTER"`, `"INNER"`, `"OPEN"`, or `"ALL"` |
| `contour_ids` | string[] | — | Explicit contour IDs within the part (overrides `contour_role`) |
| `contour_tags` | string[] | — | Apply only to contours with any of these tags |

When `contour_ids` is present, `contour_role` and `contour_tags` are ignored.

**Examples:**

```json
"target": { "scope": "ALL", "contour_role": "OUTER" }
```
→ Apply to all outer contours of all instances.

```json
"target": {
  "scope": ["inst_005", "inst_006"],
  "contour_role": "INNER"
}
```
→ Apply only to inner contours of inst_005 and inst_006.

```json
"target": {
  "contour_tags": ["thread-m8"]
}
```
→ Apply to all contours tagged `thread-m8` across all instances.

### 6.4 `toolpath_hints` Object

Describes **how** the toolpath is executed — approach geometry, cut direction, and lead-in/out. This is intentionally separate from `params` (which holds cutting physics).

| Field             | Type    | Description                                            |
| -------------------| ---------| --------------------------------------------------------|
| `approach`        | string  | `"OUTSIDE"` (default), `"INSIDE"`, `"TANGENT"`         |
| `sequence`        | string  | `"INNER_FIRST"` (default), `"OUTER_FIRST"`, `"CUSTOM"` |
| `cut_direction`   | string  | `"CLIMB"`, `"CONVENTIONAL"`                            |
| `start_position`  | string  | `"AUTO"`, `"NEAREST"`, `"BOTTOM_LEFT"`                 |
| `bridge_kerf`     | boolean | If true, leave micro-bridges (tabs) on this contour    |
| `bridge_width_mm` | number  | Micro-bridge width (mm)                                |
| `lead_in`         | object  | Lead-in geometry (see `lead_in`/`lead_out` object below) |
| `lead_out`        | object  | Lead-out geometry                                      |

**`lead_in` / `lead_out` object:**

| Field | Type | Description |
|---|---|---|
| `type` | string | `"ARC"`, `"LINE"`, `"TANGENT"`, `"NONE"` |
| `length_mm` | number | Lead length (for `LINE` and `TANGENT`) |
| `radius_mm` | number | Arc radius (for `ARC`) |
| `angle_deg` | number | Entry angle for `LINE`, arc sweep angle for `ARC` |
| `overcut_mm` | number | Extra travel past start/end point (prevents notch at contour closure) |

Different operations targeting different contours each carry their own `toolpath_hints`, enabling per-contour lead-in/out without any special syntax.

---

## 7. `toolpaths` Array (Optional)

Pre-computed toolpaths may be stored in the workspace. This allows simulation and G-code preview without re-running the CAM engine. Toolpaths are derived data — the authoritative source is the `operations` + `instances`.

| Field           | Type     | Description                           |
| -----------------| ----------| ---------------------------------------|
| `id`            | string   | Unique identifier                     |
| `operation_ref` | string   | References `operations[*].id`         |
| `instance_ref`  | string   | References `instances[*].instance_id` |
| `contour_ref`   | string   | Contour ID within the part            |
| `segments`      | object[] | Ordered toolpath moves                |

### Toolpath Segment Types

| `type` | Description | Required fields |
|---|---|---|
| `RAPID` | Non-cutting rapid move | `start`, `end`, `z` |
| `PIERCE` | Stationary pierce | `position`, `duration_ms` |
| `LEAD_IN` | Lead-in motion | `start`, `end`, `arc_center` (if arc) |
| `CUT` | Cutting move | `start`, `end`, `arc_center` (if arc), `clockwise` (if arc) |
| `LEAD_OUT` | Lead-out motion | `start`, `end`, `arc_center` (if arc) |
| `DWELL` | Pause | `duration_ms` |

---

## 8. ID and Reference Rules

1. All `id` fields within the same collection must be unique within the file.
2. `part_ref` in instances must match a `library.parts[*].id`.
3. `material_ref` in sheets must match a `library.materials[*].id`.
4. Segment IDs (`seg_xxx`) must be unique within the part.
5. Contour IDs (`contour_xxx`) must be unique within the part.
6. Instance IDs (`instance_id`) must be unique within the workspace.
7. Cross-reference resolution is always within the same file (no external references in v1.0).

---

## 9. Conformance

### 9.1 Writing

A conforming OCF writer must:
- Set `meta.ocf_version` to the OCF version it targets.
- Produce valid JSON.
- Include all **required** fields.
- Set `meta.units` to `"mm"`.
- Ensure segment chains in a closed contour form a continuous loop (tolerance ≤ 0.01 mm).

### 9.2 Reading

A conforming OCF reader must:
- Check `meta.ocf_version`. If the MAJOR version is higher than supported, report an error.
- Ignore unknown top-level keys and unknown object fields.
- Ignore operations with unknown `op_type` values (graceful degradation).
- Report a warning (not an error) when optional recommended fields are missing.
- Apply the operation priority chain (§6.1) correctly.

### 9.3 Validation

Use [schema.json](schema.json) for automated validation. All required fields and enum constraints are encoded there.

---

## 10. Extension Mechanism

### 10.1 Custom `op_type` Values

Third-party operation types must use a namespaced prefix: `VENDOR_OPERATIONNAME` (e.g., `ACME_ROTARY_ENGRAVE`). Standard names are reserved for this specification.

### 10.2 Custom Object Fields

Any object may carry additional fields prefixed with `x_` (e.g., `"x_erp_code": "MAT-001"`). These are ignored by conforming parsers.

### 10.3 Version Bump Policy

Minor additions that don't break existing parsers increment MINOR version. Any removal or incompatible change to a required field increments MAJOR version and requires a migration path in [CHANGELOG.md](../CHANGELOG.md).
