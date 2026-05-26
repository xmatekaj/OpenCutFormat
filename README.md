# OCF — Open Cut Format

**Version 1.0.0** | Open Standard for CAD/CAM Sheet-Metal Processing

OCF is an open, JSON-based file format describing the complete sheet-metal cutting workflow: part definitions, material libraries, nesting layouts, machining operations, and (optionally) computed toolpaths. A single `.ocf` file is the entire project — geometry, technology, and nesting combined.

## Supported Processes

| Category | Process Types |
|---|---|
| Thermal cutting | Laser, Plasma, Oxyfuel |
| Waterjet | Pure water, Abrasive |
| Bevel | A, V, Y, X, K profiles |
| Mechanical | Milling contour/pocket, Drilling, Tapping, Countersink |
| Punching | Punch tool, Forming, Nibbling |
| Surface | Laser marking, Engraving, Etching, Scribing |

## File Extension

`.ocf` — plain JSON text file, UTF-8 encoded.

## Minimal Valid File

```json
{
  "meta": { "ocf_version": "1.0.0" },
  "library": { "parts": [] },
  "workspaces": []
}
```

## Directory Structure

```
specification/
  OCF-v1.0.md       Master specification (top-level reading start)
  geometry.md       Coordinate system, segment types, contour rules
  operations.md     Complete operations catalog with all parameters
  schema.json       JSON Schema Draft-07 for validation

examples/
  README.md                       What each example demonstrates
  01-minimal.ocf.json             Smallest valid OCF file
  02-single-part-laser.ocf.json   One part, laser cut, no nesting
  03-nesting-laser.ocf.json       Multi-part laser nesting on full sheet
  04-plasma-bevel.ocf.json        Plasma cutting with bevel (A-profile)
  05-waterjet.ocf.json            Waterjet with abrasive parameters
  06-mixed-operations.ocf.json    Laser + drilling + tapping + marking
  07-common-line-cutting.ocf.json Common-line (bridge-cut) constraint

CHANGELOG.md        Version history and migration notes
```

## Core Design Principles

**1. Single Source of Truth**
One `.ocf` file contains everything needed to understand, reproduce, and re-process a job. No companion files, no external references required.

**2. Geometry / Machining Separation**
`library` holds clean CAD geometry. `workspaces` apply CAM operations as a separate layer. Changing a technology parameter never touches the geometry.

**3. Polymorphic Operations**
`op_type` is the extension point. Every postprocessor reads the operations it understands and safely skips the rest — a 2-axis laser machine ignores `BEVEL_CUT` without error.

**4. Non-destructive Nesting**
Original geometry in `library` is immutable. Position, rotation, and mirror are stored as instance transforms. Renesting produces a new workspace, not a modified part.

**5. Graceful Degradation**
Any conforming parser must tolerate unknown `op_type` values, unknown top-level keys, and any `meta.ocf_version` higher than it supports — emit a warning, not an error.

## Quick Concept Map

```
meta         Project metadata, version, application, author
library
  materials  Named material definitions (density, type)
  parts      Geometry definitions (contours, segments, default ops)
workspaces
  sheet      Material + thickness + dimensions
  instances  Positioned instances of library parts (transforms)
  operations Machining operations applied to instances/contours
  constraints Relationships (common line, fixed position, ...)
  toolpaths  (optional) Pre-computed toolpath for simulation
```

## Versioning Policy

OCF uses Semantic Versioning (`MAJOR.MINOR.PATCH`):
- **PATCH** — clarifications, no schema changes
- **MINOR** — new optional fields; fully backwards-compatible
- **MAJOR** — breaking schema changes; parsers must migrate

All parsers must expose the OCF version they support and reject files with a higher **MAJOR** version.

## License

OCF specification is released under [Creative Commons CC0 1.0](https://creativecommons.org/publicdomain/zero/1.0/) — public domain, no restrictions.
