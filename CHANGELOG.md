# OCF Changelog

All notable changes to the OCF specification are documented here. OCF follows [Semantic Versioning](https://semver.org/): MAJOR.MINOR.PATCH.

---

## [1.0.0] — 2026-05-26

Initial release of the Open Cut Format specification.

### Specification

**Core structure**
- `meta` object with `ocf_version`, `application`, `project`, and `units` fields
- `library` object containing `materials` and `parts` arrays
- `workspaces` array with full nesting, operations, constraints, and optional toolpaths

**Geometry model**
- Segment types: `LINE`, `ARC`, `CIRCLE`, `SPLINE` (Bézier + NURBS), `POLYLINE`
- Contour roles: `OUTER`, `INNER`, `OPEN`
- Right-handed 2D coordinate system, origin at bottom-left, units in mm, angles in degrees

**Materials**
- Material types: `STEEL`, `STAINLESS_STEEL`, `ALUMINUM`, `COPPER`, `BRASS`, `TITANIUM`, `NICKEL_ALLOY`, `PLASTIC`, `FOAM`, `WOOD`, `STONE`, `COMPOSITE`, `OTHER`

**Operations model**
- Three-level priority chain: part defaults → workspace defaults → instance overrides
- Operation targeting by scope, contour role, contour ID, or contour tags
- `override_operations` (replace) and `extend_operations` (merge) at instance level
- `toolpath_hints` per operation (approach, sequence, cut direction)

**Standard operation types**
- Thermal: `LASER_CUT`, `PLASMA_CUT`, `OXYFUEL_CUT`
- Bevel: `BEVEL_CUT` (A, V, Y, X, K profiles)
- Waterjet: `WATERJET_CUT`
- Surface: `MARKING`, `ENGRAVING`, `ETCHING`
- Drilling: `DRILLING`, `COUNTERSINK`, `TAPPING`
- Milling: `MILLING_CONTOUR`, `MILLING_POCKET`
- Punching: `PUNCH_TOOL`, `NIBBLING`, `FORMING`

**Constraints**
- `COMMON_LINE_CUTTING` (bridge/shared-edge cuts)
- `FIXED_POSITION`
- `RELATIVE_POSITION`
- `MIN_DISTANCE`
- `SAME_ROTATION`

**Toolpaths** (optional, pre-computed)
- Move types: `RAPID`, `PIERCE`, `LEAD_IN`, `CUT`, `LEAD_OUT`, `DWELL`

**Nesting parameters**
- Part spacing, edge clearance, rotation step, mirror, common lines
- Optimization targets: material utilization, cut time, travel distance
- Sequence strategies: inside-out, outside-in, left-to-right, nearest neighbour

**Statistics** (advisory, computed fields)
- Material utilization %, total cut/rapid lengths, estimated time, pierce count

**Conformance rules**
- Graceful degradation: unknown `op_type` values must be ignored, not cause errors
- Unknown JSON keys at any level must be ignored
- Files with a higher MAJOR version must be rejected with an error
- Files with the same MAJOR but higher MINOR version must be loaded (unknown fields ignored)

**Extension mechanism**
- Custom `op_type` values require vendor prefix (`VENDOR_OPNAME`)
- Custom fields may use `x_` prefix at any level

**Validation**
- JSON Schema Draft-07 (`specification/schema.json`)

---

## Migration Guide

### To v1.0.0 (baseline)

This is the initial version. There is no previous version to migrate from.

---

## Planned for future versions

The following features are candidates for v1.1 or later. They do not yet have a committed specification.

| Feature | Notes |
|---|---|
| `inch` unit support | `meta.units: "inch"` — all distances in inches |
| Remnant sheet polygons | Non-rectangular remnant sheet outlines |
| Micro-bridges (tabs) | First-class bridge definition on contours |
| Tool library section | `library.tools` for drill, mill, punch tool definitions |
| Machine parameter profiles | `library.machine_profiles` for reusable cutting parameters |
| Multi-head operations | Parallel cutting with multiple heads |
| 3D surface operations | Engraving on non-flat surfaces |
| File references | `geometry.source_ref` for linking to external DXF/STEP files |
| Simulation result data | Kerf geometry, heat-affected zone estimates in `toolpaths` |
