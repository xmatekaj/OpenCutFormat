# OCF Geometry Model

> Part of the OCF v1.0 specification. See [OCF-v1.0.md](OCF-v1.0.md) for context.

---

## 1. Coordinate System

OCF uses a right-handed, two-dimensional Cartesian coordinate system:

```
Y
↑
|
|
+--------→ X
origin
```

| Property | Value |
|---|---|
| Origin | Bottom-left corner of the part bounding box (library space) or sheet origin (workspace space) |
| X-axis | Positive to the right |
| Y-axis | **Positive upward (↑)** — **CRITICAL: differs from SVG** |
| Unit | Millimetres (mm) |
| Angle unit | Degrees (°) |
| Positive rotation | Counterclockwise (CCW) |

⚠️ **CRITICAL IMPLEMENTATION NOTE — Y-Axis Orientation & SVG Import:**

OCF uses **Cartesian coordinates (Y positive upward)**, which is standard in CNC/DXF/G-code worlds. **Standard SVG uses Y positive downward.** If you import SVG path data directly from a `.svg` file into OCF without transformation, geometry will be **inverted vertically**. 

**Full mitigation for importers from Y-down SVG to Y-up OCF:**
1. **Negate all Y-coordinates** during SVG→OCF conversion
2. **Invert sweep-flag in all `A` (Arc) commands:** 
   - SVG sweep-flag=1 (clockwise in SVG space) → OCF sweep-flag=0 (counterclockwise in Cartesian space)
   - SVG sweep-flag=0 (counterclockwise in SVG space) → OCF sweep-flag=1 (clockwise in Cartesian space)
3. **Verify orientation** by checking if contour winding (expected CCW for OUTER) matches actual geometry direction

**Example:** SVG arc `A 50 50 0 0 1 150 50` (sweep-flag=1, CW in SVG) → OCF arc `A 50 50 0 0 0 y' 50` (sweep-flag=0, CW in Cartesian after Y inversion)

### 1.1 Library Space vs Sheet Space

- In **library space** (inside `library.parts`), coordinates are part-local. The origin is the part's own reference point (typically its bounding box minimum corner).
- In **sheet space** (inside `workspaces`), coordinates are relative to the sheet origin (bottom-left corner of the sheet).
- An instance `transform` converts from library space to sheet space.

### 1.2 Transform Order & Mirror Reference Point

**Transform Sequence:**

1. Apply `mirror_x` (reflect across **bounding box center**, flip Y-axis)
2. Apply `mirror_y` (reflect across **bounding box center**, flip X-axis)
3. Apply `rotation_deg` (rotate counterclockwise around **bounding box center**)
4. Apply translation (`x`, `y`)

**Critical:** Mirror operations are anchored to the **bounding box center** of the part in library space, **not** to global origin (0,0). This prevents the part from "escaping" the viewport when mirrored.

**Bounding Box Center Calculation:**
```
bbox_center = ((min.x + max.x) / 2, (min.y + max.y) / 2)
```

**Algorithmic Implementation:**
```
1. Translate part by -bbox_center (move center to origin)
2. Apply mirror_x (negate Y)
3. Apply mirror_y (negate X)
4. Apply rotation by θ
5. Translate by +bbox_center (move center back)
6. Apply final translation (x, y) to sheet space
```

**Matrix Notation:**

```
P_sheet = Translate(x, y) × 
          Translate(bbox_center) × 
          Rotate(θ) × 
          MirrorY × 
          MirrorX × 
          Translate(-bbox_center) × 
          P_library
```

**Critical Implementation Requirement:** Before applying **any** instance transformation (`mirror_x`, `mirror_y`, `rotation_deg`), the parser **must first fully resolve the part's bounding box** in library space by:
1. Parsing all SVG path commands in the contour's `d` string
2. Computing extrema for all segments (including arc endpoints and Bézier curve extrema)
3. Deriving `bbox_center = ((min.x + max.x) / 2, (min.y + max.y) / 2)`

**Only after bounding box is known** can transformations be applied correctly. This is a hard requirement because the entire transformation matrix pivots around `bbox_center`.

**UI Implication:** When a user clicks "Mirror X" in a CAM application, the part flips around its geometric center, not the library origin. This matches intuition in design tools (Inkscape, Fusion 360, etc.).

**Concrete Example:**

Part bounding box: `min: [0, 0]`, `max: [100, 100]` → center = `(50, 50)`

Library geometry: Rectangle from (10, 10) to (90, 90)

Transform: `mirror_x: true, rotation_deg: 45, x: 200, y: 100`

Implementation:
```
1. Center at (50, 50):  (10, 10) → (-40, -40)
2. Mirror X (negate Y):  (-40, -40) → (-40, +40)
3. Rotate 45°:          (-40, +40) → (-56.6, 0) [approximately]
4. Move center back:     (-56.6, 0) → (−6.6, 50)
5. Translate to sheet:   (−6.6, 50) + (200, 100) → (193.4, 150)
```

Result: The part is mirrored, rotated, and placed at sheet position (200, 100) with its center of mass at approximately (193.4, 150).

---

## 2. Contours

A **contour** is a connected, planar path described by a standard SVG path data string.

### 2.1 Contour Roles

| Role    | Description                                                      | Closed? |
| ---------| ------------------------------------------------------------------| ---------|
| `OUTER` | External boundary of the part. Every part must have exactly one. | Yes     |
| `INNER` | Internal cutout or hole. Zero or more per part.                  | Yes     |
| `OPEN`  | Non-closed path for scoring, marking, or partial cuts.           | No      |

### 2.2 Winding Convention

The `winding` field documents the intended direction but is **purely informational metadata**.

| Role | Recommended winding | Purpose |
|---|---|---|
| `OUTER` | `CCW` (counterclockwise) | Convention to aid human readers; CAM systems verify against actual geometry |
| `INNER` | `CW` (clockwise) | Convention to aid human readers; CAM systems verify against actual geometry |

⚠️ **Parser Responsibility:** Parsers **MUST NOT trust** the `winding` field. Instead, they must:
1. Parse the SVG path (`d` string)
2. Calculate actual winding using the **Shoelace formula** (see §7.1)
3. Determine actual direction: CW or CCW
4. Use actual direction (not declared winding) for kerf compensation and tool offset calculations

**Rationale:** Incorrect `winding` metadata could cause milling machines to cut on the wrong side of a line, destroying parts. The declared `winding` field may be out-of-sync with the actual geometry direction.

### 2.3 Contour Structure

A contour is encoded as a JSON object:

```json
{
  "id": "contour_outer",
  "role": "OUTER",
  "winding": "CCW",
  "tags": ["profile"],
  "d": "M 0 0 L 100 0 L 100 75 L 0 75 Z",
  "lead_in_command_index": 1
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Unique identifier within the part |
| `role` | string | YES | `OUTER`, `INNER`, or `OPEN` |
| `winding` | string | NO | `CW` or `CCW` (metadata only; parser must verify against actual geometry) |
| `tags` | string[] | NO | Optional semantic labels |
| `d` | string | YES | SVG path data string (see §3) |
| `lead_in_command_index` | integer | NO | Zero-based SVG command index where machine should pierce/enter material. If omitted, pierce occurs at command 0 (start). |

**Lead-In Index Example:**
- `"d": "M 0 0 L 100 0 L 100 75 L 0 75 Z"` has commands [M, L, L, L, Z]
- `"lead_in_command_index": 2` means pierce at the third L command (into the top-right corner), not at M

### 2.4 Path Continuity

For **closed** contours (`OUTER` and `INNER`):
- The path must end with a `Z` (Close Path) command or have the implicit assumption that the final point connects back to the first.
- All intermediate move-to commands are forbidden (only one `M` at the start).

For **open** contours (`OPEN`):
- The path need not close. It is typically a series of move, line, and arc commands without a final `Z`.

### 2.5 Nesting of Contours

Parts may have complex topology. The OCF model:
- One `OUTER` contour defines the external boundary.
- `INNER` contours are fully contained within the `OUTER` contour.
- `INNER` contours must not overlap each other.

OCF does **not** support nested INNER contours (islands within holes). If needed, represent as separate parts.

---

## 3. SVG Path Data Syntax

OCF uses standard SVG path data syntax to encode contour geometry. The `d` field in a contour contains a sequence of **commands** and **coordinates**, all in millimetres, following the SVG specification.

### 3.1 Supported Path Commands

**Command Format Rule:** OCF permits **only uppercase commands** (absolute coordinates). Lowercase variants (m, l, a, c, q, z) indicating relative/incremental coordinates are **NOT allowed**. This eliminates ambiguity and simplifies parsers.

#### M — Move To

Moves the cursor to an absolute position without drawing.

```
M 100 200        move to absolute (100, 200)
```

⚠️ Lowercase `m` is **not permitted**.

A contour's path string typically begins with a single `M` command. Implicit line-to commands are produced by subsequent coordinates in multi-coordinate commands (allowed only within a single contour).

#### L — Line To

Draws a straight line from the current position to a new point.

```
L 100 200        line to absolute (100, 200)
```

#### H / V — Horizontal / Vertical Line To

Shorthand for lines parallel to the X or Y axis.

```
H 150            horizontal line to x=150
V 100            vertical line to y=100
```

#### A — Circular Arc To (Restricted)

Draws a **circular** arc to a destination point. **Elliptical arcs are NOT permitted.**

```
A r r 0 large-arc-flag sweep-flag x y
A 50 50 0 0 1 150 50
```

| Parameter | Type | Constraint | Description |
|---|---|---|---|
| `rx`, `ry` | number | **MUST be equal** (rx == ry) | Radius in mm. Elliptical arcs with different rx/ry are not allowed. |
| `rotation` | number | **MUST be 0** | X-axis rotation. Any non-zero value is invalid. |
| `large-arc-flag` | 0\|1 | — | 1 if arc > 180°, 0 if arc ≤ 180° |
| `sweep-flag` | 0\|1 | — | 1 for clockwise (CW), 0 for counterclockwise (CCW) |
| `x`, `y` | number | — | End point (absolute or relative) |

⚠️ **Validation Requirement:** Parsers MUST reject any `A` command where:
- `rx ≠ ry` (will cause approximation errors in CNC machines)
- `rotation ≠ 0` (elliptical arcs cannot be machined in 2D)

**Example:** A 50 mm radius circle arc from (0, 0) to (50, 50), counterclockwise, < 180°:
```
M 0 0 A 50 50 0 0 0 50 50
```

#### C / S — Cubic Bézier Curves

`C` defines a cubic Bézier segment with two control points.

```
C x1 y1 x2 y2 x y
C 25 50 75 50 100 0
```

`S` is a shorthand cubic Bézier where the first control point is inferred as the reflection of the last control point of the previous curve.

```
C 25 50 75 50 100 0 S 125 -50 150 0
```

⚠️ **Postprocessor Requirement:** CNC machines (lasers, plasma, waterjet) typically do **not** support native Bézier curves in G-code (only lines G01 and circular arcs G02/G03). 

**Postprocessors MUST linearize or approximate** cubic Bézier curves using:
- Uniform subdivision into line segments (with machine tolerance), or
- BiArc approximation (two circular arcs approximating the curve)

#### Q / T — Quadratic Bézier Curves

`Q` defines a quadratic Bézier (parabola-like) with one control point.

```
Q x1 y1 x y
Q 50 50 100 0
```

`T` is shorthand quadratic Bézier where the control point is the reflection of the previous curve's control point.

⚠️ **Postprocessor Requirement:** Same as cubic Bézier — quadratic curves must be linearized or approximated for CNC execution.

#### Z — Close Path

Closes the contour by drawing a line from the current point back to the start (`M` point). **No coordinates follow Z.**

```
Z
```

### 3.2 Closed vs. Open Contours

**Closed contours** (`OUTER`, `INNER`):
- Must end with `Z` to explicitly close, or end with a line to the start point.
- Winding convention (CW/CCW) applies.

**Open contours** (`OPEN`):
- Typically do **not** end with `Z`.
- Used for scoring, marking, or partial cuts.

### 3.3 Special Case: Standalone Circle Profile

A full circle (360°) can be represented as two semicircular arcs:

```json
{
  "id": "hole_inner",
  "role": "INNER",
  "d": "M 50 0 A 50 50 0 0 1 50 100 A 50 50 0 0 1 50 0 Z"
}
```

This creates a circle with center (50, 50) and radius 50, starting at (50, 0), drawing the upper semicircle clockwise, then the lower semicircle clockwise.

Alternatively, use `M` followed by two opposite arcs:
```
M <x> <y>
A r r 0 1 1 <x'> <y'>
A r r 0 1 1 <x> <y>
Z
```

### 3.4 Numeric Precision

- All coordinate and control-point values are in **millimetres**.
- Values may be integer or decimal (e.g., `10`, `10.5`, `-0.25`).
- Scientific notation is supported (e.g., `1e2` = 100).
- Whitespace and commas are optional delimiters; `M 100 200 L 150 250` is equivalent to `M 100,200 L 150,250`.

---

## 4. Contour ID Conventions & SVG Command Indexing

### 4.1 Contour ID Naming

While contour IDs can be any unique string, the following conventions aid readability and tracking:

| Convention | Example | Use |
|---|---|---|
| Sequential | `contour_001`, `contour_002` | Default for imports |
| Role-prefixed | `outer_profile`, `hole_1`, `hole_2` | Hand-authored files |
| Semantic | `top_edge`, `inner_cutout`, `score_line` | Self-documenting IDs |

### 4.2 Addressing Specific Path Segments

Because geometry is now unified within a single SVG path string (`d`), **constraints reference sub-segments by SVG command index**, not by individual segment IDs.

**SVG Command Index System:**
- Commands within a path are numbered starting from **0** (the `M` move command)
- Subsequent draw commands (`L`, `A`, `C`, `Q`, etc.) increment the index
- The `Z` (close) command counts as a command

**Example:** For contour `"d": "M 0 0 L 150 0 L 150 100 L 0 100 Z"`:
```
Index 0: M 0 0              (Move to origin)
Index 1: L 150 0            (Line down — bottom edge)
Index 2: L 150 100          (Line right)
Index 3: L 0 100            (Line across — top edge)
Index 4: Z                  (Close path)
```

Constraints (e.g., `COMMON_LINE_CUTTING`) use:
- `d_command_index` — zero-based index of the first command
- `d_command_count` — number of consecutive commands (optional; default 1)

Example: `"d_command_index": 1, "d_command_count": 1` references command 1 only (bottom edge).

### 4.3 Per-Segment Technology Assignment

Operations can target specific SVG commands using `command_indices` in the operation `target` (see [operations.md](operations.md#per-segment-technology-assignment-svg-command-indexing)):

```json
{
  "op_type": "LASER_CUT",
  "target": {
    "contour_id": "profile",
    "command_indices": [1, 2, 5]
  },
  "params": { "speed_mm_min": 5000, "power_pct": 80 }
}
```

This allows:
- **Different cutting speeds** on straight vs. curved segments
- **Quality degradation** on complex curves (waterjet)
- **Power/frequency tuning** for specific edges (laser)

### 4.4 Fine-Grained Targeting Without Command Indices

If you need distinct operations on individual segments:
- **Do NOT** split a single SVG command
- **Instead:** Decompose the contour into multiple contours, each with its own `d` string and ID
- Example: Split "outer_profile" into "outer_top" and "outer_bottom" if they require different cut speeds

---

## 5. Bounding Box Computation

The bounding box of a part encloses all contours in library space, computed from the parsed SVG path data.

For arcs and Bézier curves, the bounding box must include curve extrema, not just endpoints or control points. Parsers must:
1. Extract all coordinate data and control points from the SVG path string.
2. For arc commands (`A`), compute the actual curve extrema considering the radii, rotation, and sweep direction.
3. For Bézier curves (`C`, `Q`), compute extrema by finding stationary points (where the derivative is zero).

The bounding box is used for:
- Nesting initial bin placement
- Rapid collision detection
- Sheet utilisation calculation

---

## 6. DXF Import Rules

When importing a DXF file into OCF geometry, DXF entities are flattened into contours and mapped to SVG path commands:

| DXF Entity | SVG Path Command | Notes |
|---|---|---|
| LINE | `L` or `l` | Start with `M`, then `L` to end point |
| ARC | `A` or `a` | Use arc command with radii, flags, and endpoint |
| CIRCLE | Two `A` commands + `Z` | Split into two semicircular arcs or use equivalent representation |
| LWPOLYLINE | Sequence of `L` and/or `A` | Each segment depends on the bulge factor (0 = line, non-zero = arc) |
| SPLINE | `C` or `Q` commands | Extract degree (2→Q, 3→C) and control points |
| ELLIPSE | `A` with radii, rotation | Approximate as elliptical arc or decompose |
| HATCH | Ignored | Non-geometric; if needed, decompose to entity boundaries |
| TEXT / MTEXT | Ignored unless marked | Mark-layer text can be converted to `MARKING` operations |
| DIMENSION | Ignored | Non-geometric |
| BLOCK INSERT | Resolved and flattened | Recursively expand block references; apply transforms |

### 6.1 Contour Construction from DXF

1. **Group entities by layer** or connectivity.
2. **Determine role** by topology:
   - Outermost closed path → `OUTER`
   - Closed paths inside outer → `INNER`
   - Open/unclosed paths → `OPEN`
3. **Build SVG path string**:
   - Start with `M x y` (move to the first entity's start).
   - Append commands for each entity in order (ensuring continuity).
   - End with `Z` if closed, omit `Z` if open.
4. **Apply layer → tag mapping**:
   - DXF layer name → OCF contour `tags` array.
   - Standard convention: layer `CUT` or `0` → tag `[cut]`; layer `MARK` → tag `[mark]`; etc.

### 6.2 Winding and Orientation

After import, verify:
- Closed contours marked `OUTER` should have CCW winding.
- Closed contours marked `INNER` should have CW winding.
- If winding is reversed, invert all path coordinates (mirror or flip).

---

## 7. Area and Perimeter Calculation

Parsers must extract and process geometric data directly from the SVG path strings in each contour's `d` field.

### 7.1 Area Calculation

The total part area is:

```
area = area(OUTER) - sum(area(INNER_i))
```

To compute area from an SVG path:
1. Parse the path commands and accumulate coordinates.
2. Apply the **shoelace formula** (also called Gauss's area formula) for the polygon vertices.
3. For curves (arcs, Bézier segments), numerically integrate or subdivide into line segments and apply shoelace.

**Shoelace formula:**
```
area = 0.5 * |sum( (x_i * y_{i+1}) - (x_{i+1} * y_i) )|
```

For **arcs** (`A` command), decompose into line segments or use numerical integration of the parametric curve.

For **Bézier curves** (`C`, `Q`), either:
- Subdivide the curve into small line segments (adaptive subdivison), or
- Integrate the area under the curve using parametric integration.

### 7.2 Perimeter Calculation

The total perimeter (for cut-time estimation) is the sum of all contour path lengths:

```
perimeter = sum(length(path_i)) for all contours
```

For each path:
1. Sum line segment lengths: `sqrt((x2-x1)² + (y2-y1)²)`
2. For arcs, compute arc length: `length = r * θ` (where `θ` is the subtended angle in radians).
3. For Bézier curves, use numerical arc-length parameterization or subdivision.

### 7.3 Mass Estimation

Using material density:

```
mass_kg = area_mm² × thickness_mm × density_kg_m³ × 1e-9
```

The factor `1e-9` converts cubic millimetres to cubic metres.
