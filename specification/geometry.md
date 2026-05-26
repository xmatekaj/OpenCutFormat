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
| Y-axis | Positive upward |
| Unit | Millimetres (mm) |
| Angle unit | Degrees (°) |
| Positive rotation | Counterclockwise (CCW) |

### 1.1 Library Space vs Sheet Space

- In **library space** (inside `library.parts`), coordinates are part-local. The origin is the part's own reference point (typically its bounding box minimum corner).
- In **sheet space** (inside `workspaces`), coordinates are relative to the sheet origin (bottom-left corner of the sheet).
- An instance `transform` converts from library space to sheet space.

### 1.2 Transform Order

When an instance has both mirror and rotation:

1. Apply `mirror_x` (reflect across the X-axis: negate all Y coordinates)
2. Apply `mirror_y` (reflect across the Y-axis: negate all X coordinates)
3. Apply `rotation_deg` (rotate counterclockwise around the origin)
4. Apply translation (`x`, `y`)

In matrix notation:

```
P_sheet = Translate(x, y) × Rotate(θ) × MirrorY × MirrorX × P_library
```

---

## 2. Contours

A **contour** is a connected, planar path described by an ordered list of segments.

### 2.1 Contour Roles

| Role | Description | Closed? |
|---|---|---|
| `OUTER` | External boundary of the part. Every part must have exactly one. | Yes |
| `INNER` | Internal cutout or hole. Zero or more per part. | Yes |
| `OPEN` | Non-closed path for scoring, marking, or partial cuts. | No |

### 2.2 Winding Convention

The winding rule is used to distinguish material from voids:

| Role | Recommended winding | Why |
|---|---|---|
| `OUTER` | `CCW` (counterclockwise) | Standard for outer boundaries in most CAD/CAM |
| `INNER` | `CW` (clockwise) | Indicates a void in the even-odd fill rule |

Winding is **advisory** in v1.0; the `role` field is authoritative. Parsers must not rely solely on winding to determine contour role.

### 2.3 Contour Connectivity

For **closed** contours (`OUTER` and `INNER`):
- The `end` point of each segment must coincide with the `start` point of the next segment.
- The `end` point of the last segment must coincide with the `start` point of the first segment.
- Tolerance: ≤ 0.01 mm gap is considered connected. Larger gaps are a validation error.

For **open** contours (`OPEN`):
- Connectivity rule applies between consecutive segments, but the path need not close.

### 2.4 Nesting of Contours

Parts may have complex topology. The OCF model:
- One `OUTER` contour defines the external boundary.
- `INNER` contours are fully contained within the `OUTER` contour.
- `INNER` contours must not overlap each other.

OCF does **not** support nested INNER contours (islands within holes). If needed, represent as separate parts.

---

## 3. Segment Types

### 3.1 LINE

A straight line between two points.

```json
{
  "id": "seg_001",
  "type": "LINE",
  "start": [0.0, 0.0],
  "end": [100.0, 0.0]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Unique within the part |
| `type` | string | YES | `"LINE"` |
| `start` | [x, y] | YES | Start point (mm) |
| `end` | [x, y] | YES | End point (mm) |

### 3.2 ARC

A circular arc defined by start, end, and centre points. The arc travels from `start` to `end` using the specified direction.

```json
{
  "id": "seg_002",
  "type": "ARC",
  "start": [100.0, 0.0],
  "end": [150.0, 50.0],
  "center": [100.0, 50.0],
  "radius": 50.0,
  "clockwise": false
}
```

| Field       | Type    | Required | Description                                    |
| -------------| ---------| ----------| ------------------------------------------------|
| `id`        | string  | YES      | Unique within the part                         |
| `type`      | string  | YES      | `"ARC"`                                        |
| `start`     | [x, y]  | YES      | Arc start point (mm)                           |
| `end`       | [x, y]  | YES      | Arc end point (mm)                             |
| `center`    | [x, y]  | YES      | Centre of the circle (mm)                      |
| `radius`    | number  | YES      | Circle radius (mm)                             |
| `clockwise` | boolean | YES      | True = clockwise arc, False = counterclockwise |

The `radius` field is redundant (computable from `center` and `start`) but is required for robustness and validation. Parsers must verify `|start - center|` and `|end - center|` are within 0.01 mm of `radius`.

### 3.3 CIRCLE

A complete circle (360° arc). Used typically for holes and circular outer profiles.

```json
{
  "id": "seg_003",
  "type": "CIRCLE",
  "center": [50.0, 50.0],
  "radius": 10.0
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Unique within the part |
| `type` | string | YES | `"CIRCLE"` |
| `center` | [x, y] | YES | Centre of the circle (mm) |
| `radius` | number | YES | Circle radius (mm) |

A CIRCLE segment is self-contained — it must be the only segment in its contour. The contour `role` determines whether it is a cut (INNER) or profile (OUTER).

### 3.4 SPLINE

A polynomial spline curve. OCF supports Bézier curves (degree 2 or 3) and NURBS.

```json
{
  "id": "seg_004",
  "type": "SPLINE",
  "start": [0.0, 0.0],
  "end": [100.0, 0.0],
  "control_points": [
    [0.0, 0.0],
    [25.0, 50.0],
    [75.0, 50.0],
    [100.0, 0.0]
  ],
  "degree": 3
}
```

For NURBS (non-uniform rational B-splines):

```json
{
  "id": "seg_005",
  "type": "SPLINE",
  "start": [0.0, 0.0],
  "end": [100.0, 0.0],
  "control_points": [ ... ],
  "degree": 3,
  "knots": [0, 0, 0, 0, 0.5, 1, 1, 1, 1],
  "weights": [1.0, 0.7071, 1.0, 0.7071, 1.0]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Unique within the part |
| `type` | string | YES | `"SPLINE"` |
| `start` | [x, y] | YES | Curve start (must match `control_points[0]`) |
| `end` | [x, y] | YES | Curve end (must match last control point) |
| `control_points` | [[x,y], ...] | YES | Control point array (including start and end) |
| `degree` | integer | YES | Polynomial degree: 2 (quadratic) or 3 (cubic) |
| `knots` | number[] | no | Knot vector for NURBS (omit for uniform Bézier) |
| `weights` | number[] | no | Control point weights for NURBS (omit for polynomial) |

### 3.5 POLYLINE

A sequence of connected line segments encoded compactly.

```json
{
  "id": "seg_006",
  "type": "POLYLINE",
  "points": [
    [0.0, 0.0],
    [100.0, 0.0],
    [100.0, 75.0],
    [0.0, 75.0],
    [0.0, 0.0]
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | YES | Unique within the part |
| `type` | string | YES | `"POLYLINE"` |
| `points` | [[x,y], ...] | YES | Ordered point list; first = start, last = end |

A POLYLINE is equivalent to multiple LINE segments. It is a compact representation. For constraint referencing, the individual implied segments of a POLYLINE cannot be addressed by ID; use explicit LINE segments if you need `constraints` to reference individual edges.

---

## 4. Segment ID Conventions (Recommended)

While IDs can be any unique string, the following conventions aid readability:

| Convention | Example | Use |
|---|---|---|
| Sequential | `seg_001`, `seg_002` | Default for imports |
| Role-prefixed | `seg_outer_top`, `seg_hole1_arc` | Hand-authored files |
| Edge-named | `seg_right`, `seg_bottom` | Rectangles and simple profiles |

When using the `COMMON_LINE_CUTTING` constraint, the referenced segments must be LINE or ARC segments (not POLYLINE or CIRCLE), so that the shared edge is unambiguous.

---

## 5. Bounding Box Computation

The bounding box of a part encloses all contours in library space.

For ARC and SPLINE segments, the bounding box must include curve extrema, not just control points.

The bounding box is used for:
- Nesting initial bin placement
- Rapid collision detection
- Sheet utilisation calculation

---

## 6. DXF Import Rules

When importing a DXF file into OCF geometry:

| DXF Entity | OCF Segment |
|---|---|
| LINE | LINE |
| ARC | ARC |
| CIRCLE | CIRCLE (standalone contour) |
| LWPOLYLINE | POLYLINE or sequence of LINE/ARC |
| SPLINE | SPLINE (control points extracted) |
| ELLIPSE | SPLINE (approximated) |
| HATCH | Ignored (non-geometric) |
| TEXT / MTEXT | Ignored unless marking |
| DIMENSION | Ignored |
| BLOCK INSERT | Resolved and flattened |

Layer mapping: the DXF layer name is stored as a contour tag. Standard layer conventions for CAM:
- Layer `CUT` or `0` → OUTER or INNER contour (determined by topology)
- Layer `MARK` → OPEN contour, `MARKING` operation
- Layer `ENGRAVE` → OPEN contour, `ENGRAVING` operation

---

## 7. Area and Perimeter Calculation

**Area** (for mass calculation):

```
area = area(OUTER) - sum(area(INNER_i))
```

Arc area contribution uses the shoelace formula extended for curves (Green's theorem).

**Perimeter** (for cut-time estimation):

```
perimeter = sum(length(segment_i)) for all contours
```

**Mass estimation** (using material density):

```
mass_kg = area_mm² × thickness_mm × density_kg_m³ × 1e-9
```

The factor `1e-9` converts `mm³` to `m³`.
