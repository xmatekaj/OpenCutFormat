# OCF Operations Catalog v1.0

> Part of the OCF v1.0 specification. See [OCF-v1.0.md](OCF-v1.0.md) for the operations model.

This document defines all standard `op_type` values, their parameters, and their valid combinations.

---

## Overview Table

| Category | `op_type` | Description |
|---|---|---|
| Thermal cutting | `LASER_CUT` | 2D laser cutting |
| | `PLASMA_CUT` | Plasma arc cutting |
| | `OXYFUEL_CUT` | Oxyfuel flame cutting |
| Bevel | `BEVEL_CUT` | Bevelled edge (uses a base process) |
| Waterjet | `WATERJET_CUT` | Pure or abrasive waterjet |
| Surface / Decoration | `MARKING` | Line marking (laser, scribe, inkjet) |
| | `ENGRAVING` | Deep contour or area engraving |
| | `ETCHING` | Shallow surface etching |
| Drilling | `DRILLING` | Circular holes |
| | `COUNTERSINK` | Countersink or counterbore |
| | `TAPPING` | Internal threads |
| Milling | `MILLING_CONTOUR` | Profile milling |
| | `MILLING_POCKET` | Pocket / facing |
| Punching | `PUNCH_TOOL` | Single-hit punch |
| | `NIBBLING` | Multi-hit punching along a path |
| | `FORMING` | Form tool (louver, lance, rib) |
| Sheet prep | `FILM_REMOVAL` | Burn/vaporise protective film before cutting |

---

## Per-Contour Technology Assignment

**Key design rule:** `params` contains only cutting physics (power, speed, gas, pierce parameters, kerf). Lead-in/out and path strategy are NOT in `params` — they live in `toolpath_hints` on the operation.

To assign different technology to different contours, use **multiple operations of the same `op_type`**, each targeting a different contour group. This is the primary mechanism for per-contour technology in OCF:

```json
"operations": [
  {
    "op_type": "LASER_CUT",
    "target": { "contour_role": "OUTER" },
    "params": { "speed_mm_min": 2800, "power_pct": 90, "pierce_time_ms": 600 },
    "toolpath_hints": {
      "lead_in":  { "type": "ARC", "radius_mm": 5.0 },
      "lead_out": { "type": "ARC", "radius_mm": 4.0, "overcut_mm": 0.5 }
    }
  },
  {
    "op_type": "LASER_CUT",
    "target": { "contour_tags": ["bolt-hole"] },
    "params": { "speed_mm_min": 4500, "power_pct": 80, "pierce_time_ms": 180 },
    "toolpath_hints": {
      "lead_in":  { "type": "LINE", "length_mm": 1.5 },
      "lead_out": { "type": "LINE", "length_mm": 1.0, "overcut_mm": 0.2 }
    }
  },
  {
    "op_type": "LASER_CUT",
    "target": { "contour_ids": ["contour_slot_01"] },
    "params": { "speed_mm_min": 3800, "power_pct": 85, "pierce_time_ms": 300, "quality": "HIGH" },
    "toolpath_hints": {
      "lead_in":  { "type": "ARC", "radius_mm": 2.0 },
      "lead_out": { "type": "ARC", "radius_mm": 2.0, "overcut_mm": 0.3 }
    }
  }
]
```

Two operations of the same `op_type` must not target the same contour simultaneously (unless one is in `override_operations` at instance level, which replaces the other).

---

## Per-Segment Technology Assignment (SVG Command Indexing)

For advanced scenarios where different parts of a single contour require different cutting parameters (e.g., reduced speed on curves, full speed on straight lines), use **`command_indices`** in the operation target:

```json
"target": {
  "contour_id": "outer",
  "command_indices": [1, 2]
}
```

This applies the operation's `params` **only** to SVG commands at indices 1 and 2 within that contour's `d` string.

**Example:** Waterjet with quality degradation on curves:

```json
[
  {
    "op_type": "WATERJET_CUT",
    "target": {
      "contour_id": "profile",
      "command_indices": [1, 3, 5]  // Straight lines (L commands)
    },
    "params": {
      "speed_mm_min": 150,
      "quality": 4
    }
  },
  {
    "op_type": "WATERJET_CUT",
    "target": {
      "contour_id": "profile",
      "command_indices": [2, 4]  // Arc commands
    },
    "params": {
      "speed_mm_min": 80,
      "quality": 1
    }
  }
]
```

**Rules:**
- `command_indices` is **optional**. If omitted, the operation targets the **entire contour**.
- Indices are **zero-based**: 0 = `M` (Move), 1 = first draw command, etc.
- **All commands in the path MUST be uppercase** (absolute coordinates). Lowercase relative commands (m, l, a, c, q, z) are not permitted in OCF paths — see [geometry.md §3.1](geometry.md#31-supported-path-commands).
- Each command index in a contour can be targeted by **at most one operation** of the same `op_type`.
- Non-overlapping indices from different operations on the same contour are allowed.

**Implementation note:** CAM systems must parse the SVG path, extract commands by index, and apply the operation's `params` only to those segments. Parsers should validate that all commands are uppercase before indexing.

---

## `lead_in` / `lead_out` Object (used in `toolpath_hints`)

| Field | Type | Description |
|---|---|---|
| `type` | string | `"ARC"`, `"LINE"`, `"TANGENT"`, `"NONE"` |
| `length_mm` | number | Lead length (for `LINE` and `TANGENT`) |
| `radius_mm` | number | Arc radius (for `ARC`) |
| `angle_deg` | number | Entry angle for `LINE`, or arc sweep angle for `ARC` |
| `overcut_mm` | number | Extra travel past the contour start/end point (avoids notch at closure) |

---

## 1. `LASER_CUT`

2-axis laser cutting of a contour profile.

```json
{
  "op_type": "LASER_CUT",
  "target": { "contour_role": "OUTER" },
  "params": {
    "speed_mm_min": 3000,
    "power_pct": 80,
    "frequency_hz": 5000,
    "duty_cycle_pct": 100,
    "gas": "N2",
    "gas_pressure_bar": 12.0,
    "focus_offset_mm": 0.0,
    "cutting_height_mm": 0.5,
    "pierce_mode": "STATIC",
    "pierce_time_ms": 500,
    "pierce_power_pct": 30,
    "pierce_height_mm": 2.0,
    "pierce_frequency_hz": 1000,
    "kerf_mm": 0.3,
    "quality": "HIGH"
  },
  "toolpath_hints": {
    "sequence": "INNER_FIRST",
    "lead_in":  { "type": "ARC", "radius_mm": 3.0 },
    "lead_out": { "type": "ARC", "radius_mm": 2.0, "overcut_mm": 0.5 }
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `speed_mm_min` | number | mm/min | Cutting feed rate |
| `power_pct` | number | % | Laser power (0–100) |
| `frequency_hz` | number | Hz | Pulse repetition frequency (for pulsed lasers) |
| `duty_cycle_pct` | number | % | Pulse duty cycle (0–100) |
| `gas` | string | — | Assist gas: `"N2"`, `"O2"`, `"AIR"`, `"ARGON"`, `"CO2_MIX"`, `"H2"` |
| `gas_pressure_bar` | number | bar | Nozzle gas pressure |
| `focus_offset_mm` | number | mm | Focus point offset from surface (0 = surface, negative = below) |
| `cutting_height_mm` | number | mm | Nozzle standoff height during cutting |
| `pierce_mode` | string | — | `"STATIC"` (pause at pierce), `"DYNAMIC"` (moving pierce), `"RAMP"` |
| `pierce_time_ms` | number | ms | Duration of static pierce |
| `pierce_power_pct` | number | % | Laser power during pierce phase |
| `pierce_height_mm` | number | mm | Nozzle height during pierce |
| `pierce_frequency_hz` | number | Hz | Pulse frequency during pierce |
| `kerf_mm` | number | mm | Kerf width (used for contour compensation) |
| `quality` | string | — | `"FAST"`, `"LOW"`, `"MEDIUM"`, `"HIGH"`, `"MAX"` |

**Gas selection guidelines:**
- `N2` — stainless steel, aluminium (clean oxide-free edge)
- `O2` — mild steel ≥ 4 mm (exothermic assist, faster but oxidised edge)
- `AIR` — cost-saving alternative for mild steel, plastics
- `ARGON` — titanium, reactive metals (prevents oxidation)

---

## 2. `PLASMA_CUT`

Plasma arc cutting for conductive metals.

```json
{
  "op_type": "PLASMA_CUT",
  "target": { "contour_role": "OUTER" },
  "params": {
    "speed_mm_min": 2000,
    "amperage_a": 100,
    "voltage_v": 120,
    "gas": "AIR",
    "shield_gas": "AIR",
    "gas_pressure_bar": 5.5,
    "shield_pressure_bar": 4.5,
    "cutting_height_mm": 2.0,
    "pierce_height_mm": 6.0,
    "pierce_delay_ms": 500,
    "arc_voltage_control": true,
    "kerf_mm": 1.5,
    "quality": "MEDIUM"
  },
  "toolpath_hints": {
    "lead_in":  { "type": "ARC", "radius_mm": 5.0 },
    "lead_out": { "type": "LINE", "length_mm": 5.0 }
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `speed_mm_min` | number | mm/min | Cutting feed rate |
| `amperage_a` | number | A | Plasma current |
| `voltage_v` | number | V | Arc voltage reference (for AVC) |
| `gas` | string | — | Plasma gas: `"AIR"`, `"O2"`, `"N2"`, `"H35"` (Ar-H₂ 65/35), `"ARGON_H2"` |
| `shield_gas` | string | — | Shield gas type (same enum) |
| `gas_pressure_bar` | number | bar | Plasma gas pressure |
| `shield_pressure_bar` | number | bar | Shield gas pressure |
| `cutting_height_mm` | number | mm | Nozzle standoff during cutting |
| `pierce_height_mm` | number | mm | Nozzle standoff during pierce |
| `pierce_delay_ms` | number | ms | Delay between arc start and motion start |
| `arc_voltage_control` | boolean | — | Enable automatic height control (AVC/THC) |
| `kerf_mm` | number | mm | Kerf width |
| `quality` | string | — | `"ROUGH"`, `"MEDIUM"`, `"FINE"` |

---

## 3. `OXYFUEL_CUT`

Oxyfuel (oxy-acetylene, oxy-propane) flame cutting.

```json
{
  "op_type": "OXYFUEL_CUT",
  "target": { "contour_role": "OUTER" },
  "params": {
    "speed_mm_min": 400,
    "fuel_gas": "ACETYLENE",
    "oxygen_pressure_bar": 3.5,
    "fuel_pressure_bar": 0.5,
    "preheat_time_s": 30,
    "cutting_height_mm": 3.0,
    "kerf_mm": 3.0
  },
  "toolpath_hints": {
    "lead_in":  { "type": "LINE", "length_mm": 10.0 },
    "lead_out": { "type": "LINE", "length_mm": 8.0 }
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `speed_mm_min` | number | mm/min | Cutting feed rate |
| `fuel_gas` | string | — | `"ACETYLENE"`, `"PROPANE"`, `"NATURAL_GAS"`, `"LPG"` |
| `oxygen_pressure_bar` | number | bar | Cutting oxygen pressure |
| `fuel_pressure_bar` | number | bar | Fuel gas pressure |
| `preheat_time_s` | number | s | Preheat duration before cutting starts |
| `cutting_height_mm` | number | mm | Tip standoff height |
| `kerf_mm` | number | mm | Kerf width |

---

## 4. `BEVEL_CUT`

Bevelled-edge cutting using an angled cutting head. The actual cut is performed by a base process (laser, plasma, or waterjet).

```json
{
  "op_type": "BEVEL_CUT",
  "params": {
    "bevel_type": "A",
    "bevel_angle_deg": 45.0,
    "land_mm": 2.0,
    "side": "OUTER",
    "base_process": "LASER_CUT",
    "base_params": {
      "speed_mm_min": 1500,
      "power_pct": 100,
      "gas": "N2",
      "gas_pressure_bar": 14.0
    }
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `bevel_type` | string | — | Bevel profile (see table below) |
| `bevel_angle_deg` | number | ° | Bevel angle from vertical |
| `land_mm` | number | mm | Flat (vertical) land height at the bottom of the bevel |
| `side` | string | — | `"OUTER"` (bevel on outside of cut), `"INNER"` |
| `top_land_mm` | number | mm | Flat land at top (for Y and X profiles) |
| `base_process` | string | — | Underlying cut process: `"LASER_CUT"`, `"PLASMA_CUT"`, `"WATERJET_CUT"` |
| `base_params` | object | — | Overrides for the base process parameters |

**Bevel profile types:**

| `bevel_type` | Description | Cross-section |
|---|---|---|
| `A` | Single bevel (chamfer) | One angled face, one vertical land |
| `V` | Double bevel (V-groove) | Two symmetric angled faces meeting at a point |
| `Y` | Y-bevel | Single bevel with a vertical land at the top |
| `X` | Double V (X-groove) | V-bevel with a flat top land |
| `K` | K-bevel (asymmetric) | Bevel on one side, land on the other (heavy plate) |

---

## 5. `WATERJET_CUT`

Abrasive or pure waterjet cutting.

```json
{
  "op_type": "WATERJET_CUT",
  "target": { "contour_role": "OUTER" },
  "params": {
    "speed_mm_min": 500,
    "pressure_bar": 3800,
    "abrasive_type": "GARNET_80",
    "abrasive_flow_g_min": 360,
    "quality": 3,
    "pierce_mode": "DYNAMIC",
    "pierce_dwell_ms": 1000,
    "standoff_mm": 2.0,
    "kerf_mm": 1.0,
    "taper_compensation": true
  },
  "toolpath_hints": {
    "lead_in":  { "type": "LINE", "length_mm": 8.0 },
    "lead_out": { "type": "LINE", "length_mm": 5.0 }
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `speed_mm_min` | number | mm/min | Cutting feed rate |
| `pressure_bar` | number | bar | Water pressure (typical: 2000–6000 bar) |
| `abrasive_type` | string | — | `"GARNET_80"`, `"GARNET_120"`, `"GARNET_220"`, `"ALUMINUM_OXIDE"`, `"NONE"` (pure water) |
| `abrasive_flow_g_min` | number | g/min | Abrasive mass flow rate |
| `quality` | integer | — | Quality level 1–5: 1=rough/fastest, 5=fine/slowest |
| `pierce_mode` | string | — | `"STATIC"`, `"DYNAMIC"`, `"DRILL_HOLE"` |
| `pierce_dwell_ms` | number | ms | Static pierce dwell time |
| `standoff_mm` | number | mm | Nozzle standoff from surface |
| `kerf_mm` | number | mm | Kerf width |
| `taper_compensation` | boolean | — | Enable jet taper angle correction |

---

## 6. `MARKING`

Non-cutting surface marking. Does not penetrate the material.

```json
{
  "op_type": "MARKING",
  "params": {
    "method": "LASER",
    "speed_mm_min": 8000,
    "power_pct": 15,
    "frequency_hz": 20000,
    "depth_mm": 0.05,
    "line_width_mm": 0.1,
    "pattern": "SOLID",
    "text": null
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `method` | string | — | `"LASER"`, `"SCRIBE"`, `"INKJET"`, `"PUNCH_MARK"` |
| `speed_mm_min` | number | mm/min | Travel speed during marking |
| `power_pct` | number | % | Laser power (for laser marking) |
| `frequency_hz` | number | Hz | Pulse frequency (for laser) |
| `depth_mm` | number | mm | Marking depth (0 for surface-only) |
| `line_width_mm` | number | mm | Mark line width |
| `pattern` | string | — | `"SOLID"`, `"DASHED"`, `"DOTTED"` |
| `text` | string | — | Optional text string for text marking |
| `font` | string | — | Font name for text marking |
| `text_height_mm` | number | mm | Text character height |

---

## 7. `ENGRAVING`

Deep contour or area engraving. Removes material without cutting through.

```json
{
  "op_type": "ENGRAVING",
  "params": {
    "depth_mm": 0.5,
    "strategy": "RASTER",
    "line_spacing_mm": 0.15,
    "speed_mm_min": 2000,
    "power_pct": 50,
    "passes": 2
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `depth_mm` | number | mm | Target engraving depth |
| `strategy` | string | — | `"CONTOUR"` (follow the outline), `"RASTER"` (parallel lines fill), `"HATCH"` (crosshatch fill), `"SPIRAL"` |
| `line_spacing_mm` | number | mm | Raster/hatch line spacing |
| `speed_mm_min` | number | mm/min | Feed rate |
| `power_pct` | number | % | Laser/tool power |
| `passes` | integer | — | Number of depth passes |

---

## 8. `ETCHING`

Very shallow surface treatment, typically < 0.1 mm deep.

```json
{
  "op_type": "ETCHING",
  "params": {
    "depth_mm": 0.02,
    "speed_mm_min": 10000,
    "power_pct": 8,
    "frequency_hz": 50000
  }
}
```

Parameters are identical to `ENGRAVING` but `depth_mm` is typically < 0.1 mm.

---

## 9. `DRILLING`

CNC drilling of circular holes.

```json
{
  "op_type": "DRILLING",
  "params": {
    "diameter_mm": 10.0,
    "depth_mm": 6.0,
    "drill_type": "THROUGH",
    "rpm": 1500,
    "feed_mm_min": 120,
    "peck_mode": true,
    "peck_depth_mm": 2.0,
    "retract_height_mm": 5.0,
    "dwell_at_bottom_ms": 0,
    "coolant": "FLOOD",
    "spotting": {
      "diameter_mm": 3.0,
      "depth_mm": 1.5,
      "angle_deg": 90
    }
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `diameter_mm` | number | mm | Drill diameter |
| `depth_mm` | number | mm | Drill depth (0 = thickness, meaning through-hole) |
| `drill_type` | string | — | `"THROUGH"`, `"BLIND"`, `"PARTIAL"` |
| `rpm` | number | rpm | Spindle speed |
| `feed_mm_min` | number | mm/min | Feed rate |
| `peck_mode` | boolean | — | Enable peck drilling (chip breaking) |
| `peck_depth_mm` | number | mm | Peck increment |
| `retract_height_mm` | number | mm | Retract height above surface between passes |
| `dwell_at_bottom_ms` | number | ms | Dwell at hole bottom |
| `coolant` | string | — | `"FLOOD"`, `"MIST"`, `"THROUGH_TOOL"`, `"AIR_BLAST"`, `"NONE"` |
| `spotting` | object | — | Optional spotting drill operation preceding main drill |

---

## 10. `COUNTERSINK`

Countersink or counterbore for screw head seating.

```json
{
  "op_type": "COUNTERSINK",
  "params": {
    "type": "COUNTERSINK",
    "outer_diameter_mm": 14.0,
    "angle_deg": 90.0,
    "depth_mm": 3.0,
    "rpm": 1200,
    "feed_mm_min": 60,
    "coolant": "FLOOD"
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `type` | string | — | `"COUNTERSINK"` (angled), `"COUNTERBORE"` (flat-bottom) |
| `outer_diameter_mm` | number | mm | Outer diameter of the feature |
| `angle_deg` | number | ° | Included angle (for countersink; e.g. 90°, 82°) |
| `depth_mm` | number | mm | Feature depth |
| `bore_diameter_mm` | number | mm | Flat bottom diameter (for counterbore) |
| `rpm` | number | rpm | Spindle speed |
| `feed_mm_min` | number | mm/min | Feed rate |
| `coolant` | string | — | Coolant type (see `DRILLING`) |

---

## 11. `TAPPING`

Cutting internal threads.

```json
{
  "op_type": "TAPPING",
  "params": {
    "thread_designation": "M8",
    "pitch_mm": 1.25,
    "depth_mm": 12.0,
    "rpm": 300,
    "hand": "RIGHT",
    "feed_mm_min": 375,
    "coolant": "FLOOD"
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `thread_designation` | string | — | ISO thread (e.g. `"M6"`, `"M8x1.0"`) or inch (`"1/4-20 UNC"`) |
| `pitch_mm` | number | mm | Thread pitch (informational/validation) |
| `depth_mm` | number | mm | Threading depth |
| `rpm` | number | rpm | Spindle speed |
| `hand` | string | — | `"RIGHT"` (standard), `"LEFT"` |
| `feed_mm_min` | number | mm/min | Axial feed (must equal `rpm × pitch_mm`) |
| `coolant` | string | — | Coolant type |

---

## 12. `MILLING_CONTOUR`

Profile milling along a contour path.

```json
{
  "op_type": "MILLING_CONTOUR",
  "target": { "contour_role": "OUTER" },
  "params": {
    "tool_id": "EM6_HSS",
    "tool_diameter_mm": 6.0,
    "rpm": 8000,
    "feed_mm_min": 800,
    "depth_of_cut_mm": 2.0,
    "axial_passes": 3,
    "strategy": "CLIMB",
    "compensation": "LEFT",
    "finish_pass": true,
    "finish_stock_mm": 0.1,
    "finish_feed_mm_min": 400,
    "coolant": "FLOOD"
  },
  "toolpath_hints": {
    "lead_in":  { "type": "ARC", "radius_mm": 3.0 },
    "lead_out": { "type": "ARC", "radius_mm": 3.0 }
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `tool_id` | string | — | Tool identifier (from machine tool library) |
| `tool_diameter_mm` | number | mm | Tool diameter (for kerf/offset computation) |
| `rpm` | number | rpm | Spindle speed |
| `feed_mm_min` | number | mm/min | Feed rate |
| `depth_of_cut_mm` | number | mm | Axial depth per pass |
| `axial_passes` | integer | — | Number of depth passes |
| `strategy` | string | — | `"CLIMB"`, `"CONVENTIONAL"` |
| `compensation` | string | — | `"LEFT"` (G41), `"RIGHT"` (G42), `"CENTER"` (no offset) |
| `finish_pass` | boolean | — | Enable a final light finishing pass |
| `finish_stock_mm` | number | mm | Stock left for finish pass |
| `finish_feed_mm_min` | number | mm/min | Feed rate for finish pass |
| `coolant` | string | — | Coolant type |

---

## 13. `MILLING_POCKET`

Pocket or face milling of an enclosed area.

```json
{
  "op_type": "MILLING_POCKET",
  "params": {
    "tool_id": "EM6_HSS",
    "tool_diameter_mm": 6.0,
    "rpm": 6000,
    "feed_mm_min": 600,
    "depth_mm": 3.0,
    "depth_of_cut_mm": 1.0,
    "stepover_pct": 40,
    "strategy": "SPIRAL",
    "finish_pass": true,
    "finish_stock_mm": 0.1,
    "coolant": "FLOOD"
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `tool_id` | string | — | Tool identifier |
| `tool_diameter_mm` | number | mm | Tool diameter |
| `rpm` | number | rpm | Spindle speed |
| `feed_mm_min` | number | mm/min | Feed rate |
| `depth_mm` | number | mm | Total pocket depth |
| `depth_of_cut_mm` | number | mm | Axial depth per pass |
| `stepover_pct` | number | % | Radial stepover as % of tool diameter |
| `strategy` | string | — | `"SPIRAL"` (default), `"ZIGZAG"`, `"RASTER"`, `"OFFSET"` |
| `finish_pass` | boolean | — | Enable wall/floor finish pass |
| `finish_stock_mm` | number | mm | Stock left for finish pass |
| `coolant` | string | — | Coolant type |

---

## 14. `PUNCH_TOOL`

Single-stroke punching with a specific tool.

```json
{
  "op_type": "PUNCH_TOOL",
  "params": {
    "tool_id": "ROUND_D10",
    "rotation_deg": 0.0,
    "stroke_force_kn": 20.0,
    "clearance_pct": 20.0
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `tool_id` | string | — | Tool identifier (from machine tool library) |
| `rotation_deg` | number | ° | Tool rotation angle |
| `stroke_force_kn` | number | kN | Applied punch force |
| `clearance_pct` | number | % | Die clearance as % of material thickness |

---

## 15. `NIBBLING`

Multi-stroke punching along a path (for shapes larger than the punch tool).

```json
{
  "op_type": "NIBBLING",
  "params": {
    "tool_id": "RECT_20x5",
    "rotation_deg": 0.0,
    "stroke_pitch_mm": 3.0,
    "overlap_mm": 1.0,
    "stroke_force_kn": 30.0
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `tool_id` | string | — | Tool identifier |
| `rotation_deg` | number | ° | Tool orientation |
| `stroke_pitch_mm` | number | mm | Distance between successive strokes |
| `overlap_mm` | number | mm | Overlap between strokes |
| `stroke_force_kn` | number | kN | Applied punch force |

---

## 16. `FORMING`

Forming operations that deform the material without cutting through.

```json
{
  "op_type": "FORMING",
  "params": {
    "form_type": "LOUVER",
    "tool_id": "LOUVER_30x100",
    "rotation_deg": 0.0,
    "depth_mm": 5.0,
    "stroke_force_kn": 40.0
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `form_type` | string | — | `"LOUVER"`, `"LANCE"`, `"RIB"`, `"DIMPLE"`, `"KNOCKOUTS"`, `"EMBOSS"` |
| `tool_id` | string | — | Form tool identifier |
| `rotation_deg` | number | ° | Tool rotation angle |
| `depth_mm` | number | mm | Forming depth |
| `stroke_force_kn` | number | kN | Applied force |

---

## 17. `FILM_REMOVAL`

Burns or vaporises a protective film (typically polyethylene PE or PVC) applied to metal sheet surfaces before or during the cutting process. This is a laser-specific operation performed at low power with a defocused beam or a high-speed raster scan.

**When to use:**
- Sheet metal supplied with a factory-applied PE protective film (common with aluminium, stainless steel, and coated steels).
- The film must be removed in the cutting area before the main cut to prevent contamination of the kerf and lens fouling.
- Depending on the machine and film type, removal happens either across the entire part area (area mode) or only along the cut path (contour mode).

```json
{
  "op_type": "FILM_REMOVAL",
  "target": { "contour_role": "ALL" },
  "priority": 0,
  "params": {
    "method": "LASER_RASTER",
    "speed_mm_min": 15000,
    "power_pct": 8,
    "frequency_hz": 20000,
    "focus_offset_mm": 5.0,
    "strategy": "RASTER",
    "line_spacing_mm": 0.5,
    "margin_mm": 2.0,
    "passes": 1
  }
}
```

| Parameter | Type | Unit | Description |
|---|---|---|---|
| `method` | string | — | Removal method: `"LASER_RASTER"` (area scan), `"LASER_CONTOUR"` (follow cut path only), `"LASER_SPIRAL"` (spiral outward from contour), `"MANUAL"` (informational flag only) |
| `speed_mm_min` | number | mm/min | Laser head travel speed during removal |
| `power_pct` | number | % | Laser power — must be low enough to avoid marking the base metal |
| `frequency_hz` | number | Hz | Pulse frequency |
| `focus_offset_mm` | number | mm | Positive offset above focus point (defocused beam for wider spot) |
| `strategy` | string | — | Area fill strategy when `method` is `LASER_RASTER` or `LASER_SPIRAL`: `"RASTER"`, `"SPIRAL"`, `"HATCH"` |
| `line_spacing_mm` | number | mm | Distance between raster lines (must ensure full coverage) |
| `margin_mm` | number | mm | Extra clearance around the part contour or cut path to ensure full film removal |
| `passes` | integer | — | Number of passes (1 is usually sufficient for thin PE films) |

**Execution timing:**

`FILM_REMOVAL` always uses `priority: 0` (or any value lower than the cutting operations) to ensure it executes first. The cut must never precede film removal on the same contour — it would leave a charred film residue on the cut edge.

**Typical machine capability flag:**

Machines that support film removal should include `"FILM_REMOVAL"` in `machine.capabilities`. Postprocessors for machines that do not support the required method must warn the operator and skip this operation.

---

## 18. Operation Combination Rules

### Mutually exclusive on the same contour

Two operations of the same `op_type` must not target the same contour in the same workspace, unless one is in `override_operations` (which replaces the other).

### Typical operation stacks by process type

| Machine type | Common operation stack |
|---|---|
| 2D Laser (no film) | `LASER_CUT` (OUTER + INNER) + optional `MARKING` (OPEN) |
| 2D Laser (PE film sheet) | `FILM_REMOVAL` (priority 0) → `LASER_CUT` (OUTER + INNER) |
| Laser with bevel | `BEVEL_CUT` (OUTER) + `LASER_CUT` (INNER) + optional `MARKING` |
| Plasma | `PLASMA_CUT` (OUTER + INNER) |
| Waterjet | `WATERJET_CUT` (OUTER + INNER) |
| Punch-laser combo | `PUNCH_TOOL` or `FORMING` on INNER + `LASER_CUT` on OUTER |
| Punch-laser + threads | `PUNCH_TOOL` (INNER, holes) + `TAPPING` (same holes) + `LASER_CUT` (OUTER) |
| Laser + milling | `LASER_CUT` (OUTER) + `MILLING_POCKET` (INNER pocket) |

### Priority and sequence

Operations are executed in ascending `priority` order. Default execution sequence within a workspace:
1. All `INNER` contour operations (holes first)
2. All `OUTER` contour operations (profiles last)
3. `OPEN` contour operations (marking, scoring)

Override with `toolpath_hints.sequence` on individual operations.
