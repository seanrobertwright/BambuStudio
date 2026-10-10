<!-- markdownlint-disable MD013 -->

# Filament calibration sequence and the measured-cube method

Research for wayfinder ticket #5 (map: #1). Researched 2026-10-10.

Question: what is the best-practice Calibration Sequence for characterising a new
filament on a Bambu P1S with a 0.4 mm nozzle, and how is dimensional accuracy
calibrated with a measured cube?

Code references are to upstream `master` at commit `47a13eeb1` of this repo, cited as
file path plus symbol. Confidence tags: **[high]** read in primary source or code;
**[medium]** primary source but indirect, or inferred from code; **[low]** reasoned or
secondary evidence only.

---

## 1. Short answer

Recommended order for a P1S, 0.4 mm nozzle, new filament:

| # | Test | Bambu Studio entry point on a P1S | Value produced | Where the value lives |
| --- | --- | --- | --- | --- |
| 0 | Prep: dry filament, wash plate, clean nozzle, start from the closest Generic preset | n/a | n/a | n/a |
| 1 | Temperature tower | Calibration menu (developer mode) > Temperature | `nozzle_temperature`, `nozzle_temperature_initial_layer` | filament preset |
| 2 | Max volumetric speed | Calibration menu > More... > Max flowrate | `filament_max_volumetric_speed` (derated 10-20 %) | filament preset |
| 3 | Flow Dynamics (pressure advance, K) | Calibration tab > Flow Dynamics > Manual, Pattern, step 0.002 | K value | **printer** (PA profile, picked per AMS slot), not the preset |
| 4 | Flow rate | Calibration tab > Flow Rate > Manual (coarse then fine) | `filament_flow_ratio` | filament preset |
| 5 | Retraction tower (optional for PLA) | Calibration menu > Retraction test | `filament_retraction_length` (override) | filament preset |
| 6 | Cooling check (no built-in test) | judge overhangs/bridges on the temp tower | `fan_min_speed`, `fan_max_speed`, `slow_down_layer_time`, `overhang_fan_speed` | filament preset |
| 7a | Elephant foot | gauge block or 25 x 25 x 10 cube | `elefant_foot_compensation` | **process** preset (plate-specific) |
| 7b | Measured cube: XY shrink | multi-size test piece | `filament_shrink` (XY %) | filament preset |
| 7c | Measured cube: contour and hole offsets | same piece | `xy_contour_compensation`, `xy_hole_compensation` | **process** preset |
| 7d | Measured cube: Z shrink | same piece, step heights | Z % | **no Bambu Studio key** (OrcaSlicer has one) |

Why this order, in one line each:

- Temperature first: it sets viscosity, which changes max flow, K and flow ratio
  (OrcaSlicer guide; Bambu's Flow Dynamics page says to recalibrate K when temperature
  or max volumetric speed changes). **[high]**
- Max volumetric speed before K: Bambu lists an MVS change as a trigger to redo Flow
  Dynamics, and OrcaSlicer's current guide puts MVS second. **[high]**
- K before flow ratio: Bambu's Flow Rate page assumes Flow Dynamics is already done,
  and Ellis says "tune pressure advance first" before extrusion multiplier. **[high]**
- Retraction after K and flow: K reduces the retraction needed (Ellis, OrcaSlicer). **[high]**
- Dimensions last: over-extrusion, elephant foot, damp filament and untuned K all
  move measured sizes, so Bambu's XY compensation page says to fix those first. **[high]**

Cube method in five bullets:

- Do not use a single 20 mm cube. A 0.1 mm reading error on 20 mm is 0.5 %, more than
  the whole PLA shrink (CNC Kitchen). Use features of 80-100 mm, plus a small feature,
  so that scale and constant offset can be separated.
- Measure mid-face, away from corners (PA bulge), seams and the first 1-2 mm (elephant
  foot), in X and Y separately, at three heights, three readings each, after the part
  is at room temperature.
- Scale k = slope of measured vs nominal across two outer sizes. Then
  `filament_shrink` (new) = old % x k. Per-side contour error c = (measured - k x nominal) / 2,
  so `xy_contour_compensation` -= c. Per-side hole error h = (k x nominal - measured) / 2,
  so `xy_hole_compensation` += h.
- Z shrink = slope of measured vs nominal step heights, measured face-to-face rather
  than from the bed. Bambu Studio has no filament Z key; OrcaSlicer has
  `filament_shrinkage_compensation_z`.
- Check skew separately by comparing the diagonals of a large square. Skew does not
  show in cube side lengths, and neither slicer can correct it on a P1S.

---

## 2. Where the sources disagree

| Source | Order it gives |
| --- | --- |
| Bambu wiki, third-party/developer-mode calibration page (credits OrcaSlicer) | Temperature, Flow rate (coarse, fine), Pressure advance, Retraction, Max flowrate, VFA. This is the order the page presents them in, not a stated recommendation. |
| Bambu wiki, Flow Rate Calibration page | Flow Dynamics (K) before Flow Rate ("have already performed other calibrations, such as Flow Dynamics Calibration") |
| OrcaSlicer wiki, current Calibration Guide | Temperature, Max Volumetric Speed, Pressure Advance (and Adaptive PA), Flow, Retraction, Cornering, Input Shaping, VFA. Tolerance is listed separately. |
| Ellis' Print Tuning Guide ("Read/follow these in order") | Extruder calibration, build surface prep, first layer squish, pressure advance, extrusion multiplier, PA/EM oddities, cooling and layer times, retraction, infill/perimeter overlap, stepover. Max volumetric flow sits under "Advanced Tuning". |

Points of disagreement and how this note resolves them:

1. **Flow ratio before or after K.** Bambu's developer-mode page lists flow first. Bambu's
   own Flow Rate page, Ellis and the current OrcaSlicer guide put K first. Ellis'
   argument: PA "only changes the distribution of material, not the amount", so its
   errors show as corner bulges and gaps that confuse a top-surface flow test. Resolution:
   K first, then flow; repeat K once if flow moved by more than a few percent.
   **[medium]** (the iterate-once advice is reasoning, not a source).
2. **Where max volumetric speed goes.** OrcaSlicer puts it second; Ellis treats it as
   advanced; Bambu's developer-mode page lists it near the end. Bambu's Flow Dynamics page
   names an MVS change as a reason to recalibrate K, which supports doing MVS before K.
   **[high]** for the Bambu statement.
3. **Extruder (e-steps) and first-layer squish.** Ellis starts with these. Bambu Studio
   defines no e-steps or Z-offset option (neither appears in `PrintConfig.cpp`), and the
   P1S levels its own bed. These steps drop out of the sequence; elephant foot
   compensation stands in for first-layer squish. **[medium]**
4. **Cooling.** Only Ellis has a cooling step, placed before retraction. Neither Bambu
   Studio nor OrcaSlicer ships a cooling calibration test. **[high]**
5. **Cornering, input shaping, VFA.** OrcaSlicer has tests for these. On Bambu printers
   motion tuning is done by the firmware, and VFA is a printer test, not a filament one.
   They are not part of a filament sequence. **[medium]**
6. **Calipers for flow.** Ellis cites Prusa's knowledge base and Teaching Tech as
   setting flow by measuring one or two walls, and calls this a misconception: a 0.02 mm
   reading error on a 0.4 mm wall is about 5 %. Bambu and OrcaSlicer both use visual top-surface tests. Resolution:
   never set flow from dimensions. **[high]**

---

## 3. Per-test detail

### 3.1 Temperature tower

- **Measures:** the print temperature that gives the best balance of stringing, layer
  adhesion, overhangs and bridging. **[high]** (Bambu developer-mode page; OrcaSlicer temp page)
- **Bambu Studio:** `Temp_Calibration_Dlg` in `src/slic3r/GUI/calib_dlg.cpp`. It defaults
  to 230 down to 190 C, and the step field is fixed at 5 C. In `GCode.cpp`, inside the
  `CalibMode::Calib_Temp_Tower` branch, the temperature changes every 10 mm of height.
  **[high]**
- **Produces:** `nozzle_temperature`, `nozzle_temperature_initial_layer`, and a sanity
  check of `nozzle_temperature_range_low/high`.
- **Advice:** if several blocks look good, take the middle of the range. Go toward the
  top of the range if you will print fast (OrcaSlicer). **[high]**
- **Dependence:** mostly the filament product. Faster printing (higher flow) wants
  hotter, so the result is tied to the hotend's melt capacity. Bambu's own presets use
  one temperature across nozzle sizes and nozzle flow types (for example
  `Bambu PLA Basic @BBL P1S 0.4 nozzle` and `0.6 nozzle` are both 220 C). No primary
  source found on colour shifting the optimum temperature. **[medium]**

### 3.2 Max volumetric speed

- **Measures:** the flow (mm^3/s) at which extrusion starts to fail on a speed ramp.
  Read off the height where quality drops: `start + height x step`. **[high]**
- **Bambu Studio:** the developer-mode menu item "Max flowrate"
  (`MaxVolumetricSpeed_Test_Dlg`, defaults 5 to 20 mm^3/s). The Calibration tab's MVS
  wizard is commented out: `CalibrationPanel` builds only `PressureAdvanceWizard` and
  `FlowRateWizard`. **[high]**
- **Produces:** `filament_max_volumetric_speed`. Derate it: Bambu says 5-10 %,
  OrcaSlicer says 10-20 %, because the test is a best case without retractions.
  **[high]**
- **Dependence:** strong on hotend and nozzle, and on filament. OrcaSlicer: "Even for
  the same material type (e.g., PLA), the brand and color can significantly affect the
  maximum flow rate." **[high]** Bambu's P1S PLA Basic preset carries 21 mm^3/s for the
  "Direct Drive Standard" variant and 29 for "Direct Drive High Flow". Its 0.2 mm nozzle
  preset carries 2. **[high]**

### 3.3 Flow Dynamics (pressure advance, K)

- **Measures:** how much extrusion to lead and lag during acceleration so that corners
  are neither bulged nor starved. **[high]**
- **P1S specifics:** "P1 supports only manual calibration". P1 printers do not have the
  "Flow Dynamics Calibration" option at print start. Bambu recommends the Pattern method
  with step 0.002, and calls straight lines "less accurate". **[high]** (Bambu Flow
  Dynamics page). Ellis also recommends the pattern method and has deprecated lines. **[high]**
- **Where K lives (important for the record design):** for Bambu printers the result is
  saved **to the printer**. `PressureAdvanceWizard::on_cali_save` calls
  `CalibUtils::set_PA_calib_result` or `command_extrusion_cali_set`. A
  `PACalibResult` (`src/libslic3r/Calib.hpp`) is keyed by `filament_id`, `setting_id`,
  `nozzle_diameter` and `nozzle_volume_type` (plus extruder and nozzle position for
  multi-nozzle printers). The filament preset's `pressure_advance` is never emitted for
  Bambu printers: in `GCode.cpp` the `set_pressure_advance` calls are guarded by
  `!is_BBL_Printer()` with the comment "BBS: never use for Bambu Printer", and the
  option tooltip says "Useless for Bambu Printer". **[high]**
- **Dependence:** printer plus filament ("The K value is specific to a printer and
  filament combo", Bambu). Redo it after a nozzle change, when the nozzle wears, or when
  MVS or temperature changes (Bambu). Ellis adds nozzle size, large temperature changes
  and hotend or extruder changes, and says colour "usually" is not significant. Bambu
  reports about 10 % run-to-run jitter. **[high]**

### 3.4 Flow rate (flow ratio)

- **Measures:** the extrusion multiplier that gives the smoothest top surface without
  gaps. **[high]**
- **P1S specifics:** manual only ("Only X1 series support Auto calibration"). Coarse
  covers 80-120 % in 5 % steps; fine covers 91-100 % in 1 % steps. Bambu advises against
  picking a sample with faint under-extrusion. **[high]** (Bambu Flow Rate page)
- **Bambu Studio:** Calibration tab `FlowRateWizard`. It saves `filament_flow_ratio` into
  a new filament preset (`CalibrationWizard::save_preset`, `save_preset_with_index`).
  The developer-mode menu has Coarse and Fine (Pass 1 / Pass 2) projects (`Plater::calib_flowrate`):
  new = old x (100 + modifier) / 100. OrcaSlicer also has "YOLO" methods (new = old +/-
  modifier) that Bambu Studio lacks. **[high]**
- **Dependence:** per filament brand and type; "may vary by color or by spool" (Ellis).
  Bambu's P1S PLA Basic preset carries 0.98 for the standard nozzle and 0.985 for High
  Flow, so the hotend variant matters slightly. **[high]**

### 3.5 Retraction

- **Measures:** the shortest retraction length that gives a clean tower. **[high]**
- **Bambu Studio:** `Retraction_Test_Dlg` (developer-mode menu). Produces
  `filament_retraction_length`, a per-filament override of the printer's
  `retraction_length` (listed in `filament_extruder_override_keys` in `PrintConfig.cpp`).
  **[high]**
- **Notes:** for low-ooze PLA or ABS the tower may be clean from the start; use
  0.2-0.4 mm. Persistent stringing usually means damp filament or a leaking nozzle, not
  retraction (Bambu and OrcaSlicer pages, word for word the same). Ellis: tune PA first,
  start at 0.5 mm on direct drive and stay under 1 mm. Bambu's P1S PLA Basic preset sets
  0.4 mm for the High Flow variant. **[high]**
- **Dependence:** extruder and hotend path, filament stringiness and moisture. Not
  colour. **[medium]**

### 3.6 Cooling

- **No calibration test exists** in Bambu Studio or OrcaSlicer. **[high]** Ellis: for
  PLA, more cooling is better; keep fan speed constant to avoid banding; use a minimum
  layer time (15 s for ABS). **[high]**
- **Produces (by judgement):** `fan_min_speed`, `fan_max_speed`, `fan_cooling_layer_time`,
  `slow_down_layer_time`, `overhang_fan_speed`, `close_fan_the_first_x_layers`,
  `additional_cooling_fan_speed` (all filament keys in `PrintConfig.cpp`). **[high]**
- **Dependence:** printer fan hardware and enclosure. Bambu's A1 and H2D PLA Basic
  presets override `fan_min_speed` and `fan_max_speed`; the X1C and P1S presets inherit
  them unchanged. **[medium]**

### 3.7 Dimensional (section 4)

---

## 4. The measured-cube method

### 4.1 Why a plain 20 mm cube is not enough

- **Error ratio.** "A 0.1 mm measuring error on a 20 mm cube is 0.5 %" (CNC Kitchen). Typical
  shrink is the same size or smaller: CNC Kitchen quotes about 0.35 % for PLA, 0.4 % for PETG
  and about 0.7 % for ABS and ASA. Prusa ships 0.18-0.28 % for ABS, ASA, PC and PA on the
  Core One. Ellis says 100-101 % XY scaling is the range to expect with ABS. **[high]**
- **Scale versus offset.** An outer face measures true size x scale plus the wall bulge
  on both sides. One size gives one equation with two unknowns. Measuring inside and
  outside, or two sizes, separates them (CNC Kitchen's inner and outer averaging).
  **[high]** for the method; the formulas below are this note's derivation. **[medium]**
- **Skew.** A cube "tell[s] you nothing about skew" (CNC Kitchen). Skew shows as unequal
  diagonals of a square (Klipper's method uses diagonals AC and BD plus side AD). **[high]**
- **Elephant foot and corners** spoil whichever face you measure (CNC Kitchen; Bambu's
  elephant-foot page says to avoid corners because of Flow Dynamics). **[high]**
- **Bambu's own wiki** does suggest a plain 20 mm or 100 mm cube and the formula
  measured / designed x 100 %. Its worked example (20 mm printing as 18.5 mm, so 92.5 %)
  is about 20 times larger than real material shrink. Treat that example as illustrative
  only. **[medium]**

### 4.2 Suggested test piece (for the new Calibration Test)

This is a recommendation built from the sources, not a published standard. **[medium]**

- A square **frame**, about 100 x 100 mm outside and 80 x 80 mm inside, 10-15 mm tall.
  It gives a large outer contour and a large hole in each axis.
- A **small boss** (about 20 x 20 mm) and a **small square hole** (about 10 x 10 mm)
  on the same plate. These are the second size for each fit.
- **Steps for Z:** two or three plateaus (for example at 5, 15 and 35 mm), so that Z is
  measured face to face and not from the bed.
- **A 0.5-1 mm 45-degree chamfer** on the bottom outer edge and in the bottom of the
  holes, so that elephant foot does not reach the measuring faces (the CaliFlower uses
  chamfers for this).
- **Seam** set to a corner or the back face so that it never sits mid-face.
- **One round hole** (for example 10 mm), only to check against the square-hole result.
  Round holes are polygonised, and caliper jaws read a round hole short.
- Print it with the user's normal process preset (walls, infill, speeds). Bambu: "different
  structures show different errors". **[high]**
- Calibrate `elefant_foot_compensation` first, using Bambu's gauge block or a
  25 x 25 x 10 mm cube. Its value is plate-specific (Bambu). **[high]**

### 4.3 What to measure, and how often

- **X and Y separately.** Do not average X with Y: if they differ, that is the machine,
  not the material (section 4.6).
- **Mid-face only.** Stay away from corners, which bulge because of K, and from the seam.
  **[high]** (Bambu elephant-foot page; Ellis: "measuring flat edges - not corner bulging
  or seams")
- **Height.** Measure above the chamfer, or at least 1-2 mm above the bed. Take three
  heights per face: low, middle and high.
- **Repeats.** Three readings at each point, averaged. CNC Kitchen measures each of 10
  reference dimensions three times, in about 5 minutes. Klipper needs 3 lengths per
  plane for skew. If the readings at one point spread by more than about 0.05 mm,
  re-seat the part and measure again. **[medium]** for the threshold.
- **Skew check.** Measure both diagonals of the 100 mm frame. With skew they differ;
  CNC Kitchen calls under 0.1 degrees normal. **[high]**
- **Timing.** Measure at room temperature after the bed has cooled, and record how long
  after printing (Bambu: measure after it cools to room temperature). Nylons change size
  with moisture, so measure them soon after printing. **[medium]**

### 4.4 Tools

- **Digital calipers** with 0.01 mm resolution are fine for features of 50-100 mm. The
  relative error falls as the feature grows: CNC Kitchen's 0.1 mm error is 0.5 % on 20 mm
  but 0.1 % on 100 mm. Use steady, light pressure; CNC Kitchen uses calipers with a thumb
  wheel and keeps the jaws parallel to the part. **[high]**
- **Inside jaws** under-read small round holes. Use gauge pins, drill shanks or a known
  screw for holes. Bambu's hole method uses screw fit and notes that an M6 screw is
  about 5.9 mm in practice. **[medium]**
- **A micrometer** is better for small outer features. Bambu's elephant-foot page asks for
  a caliper or micrometer with at least 0.01 mm resolution. **[high]**
- **Fit gauges** (screw, spool hub, Allen key) are a calipers-free alternative: Bambu's
  hole and contour pages, and OrcaSlicer's tolerance test with a 6 mm Allen key. They
  give a go/no-go fit, not a number to put in a record. **[high]**

### 4.5 Computation

Per axis, model the printed size of an outer feature of nominal size L and a hole of
nominal size D as:

```text
O(L) = k * L + 2c      k = scale (true size / nominal, for example 0.996)
H(D) = k * D - 2h      c = per-side outward bulge of outer contours (mm)
                       h = per-side inward bulge into holes (mm)
```

From a print made with the current settings (`p0` = current `filament_shrink` %,
`cc0` = current `xy_contour_compensation`, `hc0` = current `xy_hole_compensation`):

```text
k   = (O2 - O1) / (L2 - L1)                 slope across two outer sizes
c   = (O1 - k * L1) / 2                     residual per-side contour error
h   = (k * D1 - H1) / 2                     residual per-side hole error (repeat for D2 as a check)

filament_shrink (new)          = p0 * k     in %, for example 100 % * 0.99625 = 99.6 %
xy_contour_compensation (new)  = cc0 - c
xy_hole_compensation (new)     = hc0 + h
```

Worked example: L1 = 100 measures 99.75, L2 = 20 measures 20.05, D1 = 80 measures
79.50 and D2 = 10 measures 9.76. Then k = 79.70 / 80 = 0.99625, c = 0.0625 and
h = 0.10 at both hole sizes. Enter `filament_shrink` 99.6 %,
`xy_contour_compensation` -0.06 and `xy_hole_compensation` +0.10.

- **Why `p0 * k`:** Bambu Studio scales slices by 100 / p
  (`PrintObject::slice_volumes`: `scale = 1 / (filament_shrink * 0.01)`). A part printed
  at p0 that still measures k x nominal therefore needs p = p0 x k. Re-measuring a
  compensated print and using measured / designed on its own (the Bambu wiki formula)
  is only right when p0 is 100 %. **[high]** (code) / **[medium]** (derivation)
- **If the hole slope disagrees with the outer slope** by more than measurement noise,
  trust the outer fit for k and look for a cause: under-sized small holes from
  polygonisation, or seam bulges. **[low]**
- **With only one frame** (one outer and one inner size), assume c = h and use
  k = (O + H) / (L + D). This is CNC Kitchen's inner and outer averaging. **[medium]**
- **Z:** k_z = (Z2_meas - Z1_meas) / (Z2_nom - Z1_nom), measured plateau to plateau.
  Use plateau heights that are whole multiples of the layer height. Bambu Studio's
  `precise_z_height` adjusts the last layers to hit exact heights. **[medium]**
- **Elephant foot:** flare per side = (bottom width - reference width) / 2. Bambu's wiki
  instead averages the full-width differences, (dX + dY) / 2. But the code applies the
  value as an inward offset on each edge: in `ElephantFootCompensation.cpp`
  `elephant_foot_compensation`, `min_contour_width_compensated` = min + 2 x compensation.
  So the wiki's number may over-shrink the first layer by up to 2 times. Start from the
  per-side value and verify with a print. **[medium]**

### 4.6 Mapping to Bambu Studio settings and how the code applies them

| Setting | Preset | Definition (from `src/libslic3r/PrintConfig.cpp`) | How it is applied |
| --- | --- | --- | --- |
| `filament_shrink` | filament (`s_Preset_filament_options`) | coPercents, default 100 %, min 10. "94% if you measure 94mm instead of 100mm ... scaled in xy ... Only the filament used for the perimeter is taken into account ... compensation is done after the checks." | `PrintObject::slice_volumes` (`PrintObjectSlice.cpp`) scales every region's slices by 1 / (shrink / 100), using the region's **wall filament**. `ExPolygon::scale` scales the contour **and its holes**. It is skipped, with a critical warning, for colour-painted multi-filament objects (`should_skip_filament_shrink`). |
| `xy_contour_compensation` | **process** (`s_Preset_print_options`) | mm; "Positive value makes contour bigger." | Per-side offset applied after the shrink scaling (`_shrink_contour_holes`). Zeroed for colour-painted multi-filament objects; Bambu Studio also warns that it is not used on fuzzy-skin-painted objects. Can be overridden per object (`GUI_Factories.cpp`). |
| `xy_hole_compensation` | **process** | mm; "Positive value makes holes bigger." | Per-side offset of holes; the diameter changes by twice the value (Bambu wiki). Applies only to closed holes in a layer. |
| `elefant_foot_compensation` | **process** | mm; "Shrink the initial layer on build plate". | First layer only, and only when `raft_layers == 0`. |
| `enable_circle_compensation` + `circle_compensation_manual_offset` | **process** | "Auto circle contour-hole compensation", for circles under 50 mm; the tooltip names PLA Basic, PLA CF, PET CF, PETG CF and PETG HF | `LayerRegion::auto_circle_compensation`: per-side offset = coef_1 x speed + coef_2 x diameter + coef_3, clamped to limits. The coefficients are **filament** keys (`counter_coef_1..3`, `hole_coef_1..3`, `counter_limit_*`, `hole_limit_*`, `diameter_limit`). Turning it on zeroes the manual XY contour and hole values (`ConfigManipulation`). |

Other code facts that matter **[high]**:

- `ConfigManipulation` resets any `xy_hole_compensation` or `xy_contour_compensation`
  with an absolute value over 2 mm to 0, with a "use model scaling" dialog.
- Shipped BBL filament presets leave `filament_shrink` at 100 % (it is set only in
  `fdm_filament_common.json`).
- Tuned auto-circle coefficients ship only for H2C, H2D, H2DP, H2S, P2S and X2D filament
  presets. A P1S falls back to the generic defaults. Bambu's wiki says the feature is
  "mainly optimized for the H2D".
- **Bambu Studio has no Z shrink key and no skew correction key.** Neither
  `filament_shrinkage_compensation_z` nor any `*skew*` option is defined in
  `PrintConfig.cpp`.
- `filament_shrink` is one value for both X and Y. If X and Y scale differently, the
  difference cannot be stored in it; only per-object X and Y scaling can correct it.

### 4.7 Pitfalls

1. **Elephant foot** inflates any measurement taken near the bed. Chamfer or measure
   higher, and calibrate it first. It is plate-specific (Bambu). **[high]**
2. **Corner bulge and seams** inflate outer dimensions. Measure mid-face. **[high]**
3. **Flow ratio and K set wrong** shift every dimension. Finish steps 3-4 before
   measuring. Never tune flow from dimensions (Ellis). **[high]**
4. **One size cannot separate scale from offset.** Use two sizes, or inside and outside.
   **[high]**
5. **XY skew is not shrink.** Equal X and Y sides can hide skew. Check the diagonals.
   On a P1S there is no slicer-side fix in Bambu Studio or OrcaSlicer; Klipper advises
   fixing a badly skewed machine mechanically first. **[high]**
6. **X and Y disagreeing** points at the machine (belt tension or calibration), not the
   material. Ellis: "Don't mess with your steps_per_mm ... Deviations are almost always
   from material shrinkage, bulging, layer inconsistencies". **[high]**
7. **Iteration.** Multiply the old value by k rather than replacing it (section 4.5).
   **[medium]**
8. **Porting values between slicers.** PrusaSlicer stores shrink as the deficit (enter 1 %
   if 100 mm measures 99 mm; scale = 100 / (100 - p)). Bambu Studio and OrcaSlicer store
   the measured ratio (99 %; scale = 100 / p). The exact conversion is
   p_bambu = 100 - p_prusa. **[high]** (PrusaSlicer `filament_shrinkage_compensation_xy`;
   OrcaSlicer `Print::shrinkage_compensation`)
9. **The wiki and the code disagree in three places.** **[medium]**
   - Bambu's shrinkage page says shrinkage "does not affect the size of internal
     features (such as holes)". The code scales holes as well (`ExPolygon::scale`).
   - The `circle_compensation_manual_offset` tooltip says positive means tighter. The code
     subtracts manual_offset / 2 from the contour offset (positive shrinks shafts, a
     looser fit), and the wiki says positive makes the fit looser.
   - The elephant-foot formula differs (section 4.5).
10. **Geometry and environment.** Shrink varies with part size, wall and infill count,
    chamber temperature (P1S lid and door) and cooling. Prusa ships non-zero shrink only
    for its enclosed Core One, which shows the value is printer-dependent. **[high]**
11. **Fibre-filled filaments** shrink much less in XY. Prusa's CF variants carry 0 % XY
    but a non-zero Z value; Bambu's shrinkage page makes the same point. **[high]**
12. **Z shrink sources disagree.** Ellis: "Shrinkage occurs much less in the Z axis."
    Prusa ships Z equal to XY for ABS, ASA, PC and PA on the Core One. Z readings also
    carry first-layer squish and layer-height rounding, so record Z only when the
    face-to-face slope is clearly outside noise. **[medium]**

---

## 5. OrcaSlicer compared with Bambu Studio

| Feature | OrcaSlicer (`main` @ `07a12f5`) | Bambu Studio (`master` @ `47a13eeb1`) |
| --- | --- | --- |
| Z shrinkage | `filament_shrinkage_compensation_z` ("Shrinkage (Z)", 50-150 %) | none |
| How shrink is applied | whole-object instance scaling in X, Y and Z (`Print::shrinkage_compensation`, applied in `PrintApply.cpp` `print_objects_from_model_object`). Applied only when all used filaments have the same shrink (`has_same_shrinkage_compensations`). | per-region slice scaling in XY by the wall filament; skipped on colour-painted multi-filament objects |
| Calibration menu | Temperature, Max flowrate, Pressure advance (line, pattern, tower; adaptive PA), Flow ratio (YOLO and 2-pass), Retraction, Cornering, Input shaping, VFA, plus a "Calibration Guide" link; menu order follows its guide | same family of tests, from SoftFever/OrcaSlicer, behind **developer mode** for Bambu printers (`MainFrame::update_calibration_button_status`). No YOLO flow, cornering, input shaping or adaptive PA. The Calibration tab has the Flow Dynamics and Flow Rate wizards. |
| Tolerance test | "Orca Tolerance Test" handy model (hexagon holes with stepped clearances, 6 mm Allen key) | none built in; the wiki points to MakerWorld models |
| Precision options | Precise wall and Precise Z height exposed in its tolerance guide | `precise_z_height` is exposed; `precise_outer_wall` exists but is `comDevelop` (hidden) |
| Elephant foot | `elefant_foot_compensation` plus `elefant_foot_compensation_layers` (tapers over N layers) | single-layer `elefant_foot_compensation` only |
| Auto circle compensation | not present (no `enable_circle_compensation` in its `PrintConfig.cpp`) | present, with filament-level coefficients |
| Skew correction | none | none |

Sources for this table: OrcaSlicer `src/libslic3r/PrintConfig.cpp`, `src/libslic3r/Print.cpp`,
`src/libslic3r/calib.hpp` `CalibMode`, `src/slic3r/GUI/MainFrame.cpp`, and the OrcaSlicer
wiki tolerance page. **[high]**

---

## 6. Implications for the map

- **K is not a preset value on a P1S.** A Calibration Record can store K, but applying
  it "at slice time" through the filament preset does nothing on Bambu printers. It must
  be pushed to the printer's PA profile list (keyed by `filament_id`, nozzle diameter and
  nozzle volume type) and selected on the AMS slot, or the fork must emit `M900` itself,
  which upstream deliberately does not do. **[high]**
- **Contour, hole and elephant-foot values are process keys,** but they are
  filament-dependent. A per-filament record therefore has to inject process overrides at
  slice time, or reuse the filament-level auto-circle coefficients (circles only, needs
  the process toggle), or add a filament-level key. **[high]** for the facts;
  the options are this note's.
- **The nozzle key needs nozzle flow type.** Bambu's own P1S presets carry different flow
  ratio and MVS values for "Direct Drive Standard" and "Direct Drive High Flow" at the
  same 0.4 mm diameter, and `PACalibResult` keys on `nozzle_volume_type`. **[high]**
- **Elephant foot is per plate,** which is not part of the record's key. **[high]**
- **Z shrink has nowhere to go** in Bambu Studio. **[high]**

## 7. New questions this research raises

1. Should the Calibration Record key be printer model x nozzle diameter x **nozzle flow
   type** (Standard / High Flow), rather than printer model x nozzle diameter?
2. How does a record's K reach a P1S: by writing a printer-side PA profile through the
   existing calibration-result API, by the fork emitting `M900 K` in filament start
   G-code, or by recording it only?
3. How are per-filament contour, hole and elephant-foot values applied, given they are
   process keys: slice-time process override, a new filament key, or the auto-circle
   coefficients?
4. Is elephant-foot compensation part of the record, keyed additionally by plate type,
   or a process value outside the record?
5. Should the measured-cube test record Z shrink even though Bambu Studio cannot apply
   it, or should the fork add a filament Z-shrink key (an OrcaSlicer port)?
6. Should the sequence ask the user to turn on developer mode (needed for the temperature,
   MVS and retraction tests on Bambu printers), or should the fork surface those tests
   another way?

---

## Sources

Bambu Lab (primary, vendor):

- Bambu Studio calibration for third-party printers / developer mode:
  <https://wiki.bambulab.com/en/bambu-studio/Calibration>
- Flow Rate Calibration: <https://wiki.bambulab.com/en/software/bambu-studio/calibration_flow_rate>
- Flow Dynamics Calibration: <https://wiki.bambulab.com/en/software/bambu-studio/calibration_pa>
- X-Y Hole/Contour Compensation: <https://wiki.bambulab.com/en/software/bambu-studio/xy-hole-contour-compensation>
- 3D prints shrinkage: <https://wiki.bambulab.com/en/knowledge-sharing/3d-prints-shrinkage>
- Elephant foot: <https://wiki.bambulab.com/en/software/bambu-studio/parameter/elephant-foot>
- Auto circle contour-hole compensation: <https://wiki.bambulab.com/en/software/bambu-studio/manual/auto-circle-contour-compensation>

Bambu Studio code (this repo, upstream `master` @ `47a13eeb1`):

- `src/libslic3r/PrintConfig.cpp`: definitions of `filament_shrink`, `xy_contour_compensation`,
  `xy_hole_compensation`, `elefant_foot_compensation`, `enable_circle_compensation`,
  `circle_compensation_manual_offset`, `counter_coef_*`, `hole_coef_*`, `precise_z_height`,
  `pressure_advance`, `filament_flow_ratio`, `filament_max_volumetric_speed`;
  `filament_extruder_override_keys`
- `src/libslic3r/Preset.cpp`: `s_Preset_print_options`, `s_Preset_filament_options`
- `src/libslic3r/PrintObjectSlice.cpp`: `should_skip_filament_shrink`, `PrintObject::slice_volumes`
- `src/libslic3r/ExPolygon.cpp`: `ExPolygon::scale`
- `src/libslic3r/ElephantFootCompensation.cpp`: `elephant_foot_compensation`
- `src/libslic3r/LayerRegion.cpp`: `LayerRegion::auto_circle_compensation`
- `src/libslic3r/GCode.cpp`: `is_BBL_Printer()` guards around `set_pressure_advance`;
  `CalibMode::Calib_Temp_Tower` branch
- `src/libslic3r/Calib.hpp`: `CalibMode`, `PACalibResult`
- `src/slic3r/GUI/CalibrationPanel.cpp`: wizard list; `src/slic3r/GUI/CalibrationWizard.cpp`:
  `PressureAdvanceWizard::on_cali_save`, `CalibrationWizard::save_preset`
- `src/slic3r/GUI/MainFrame.cpp`: calibration menu, `MainFrame::update_calibration_button_status`
- `src/slic3r/GUI/ConfigManipulation.cpp`: XY compensation limits and the circle-compensation interlock
- `src/slic3r/GUI/calib_dlg.cpp`: test dialog defaults
- `resources/profiles/BBL/filament/Bambu PLA Basic @BBL P1S 0.4 nozzle.json`,
  `fdm_filament_template_direct_dual.json`, `fdm_filament_common.json`;
  `resources/profiles/BBL/machine/Bambu Lab P1S 0.4 nozzle.json` (`extruder_variant_list`)

OrcaSlicer (primary, wiki and code):

- Calibration guide: <https://www.orcaslicer.com/wiki/guides/calibration_guide.html>
- Temperature: <https://www.orcaslicer.com/wiki/calibration/temp_calib.html>
- Max volumetric speed: <https://www.orcaslicer.com/wiki/calibration/volumetric_speed_calib.html>
- Pressure advance: <https://www.orcaslicer.com/wiki/calibration/pressure_advance_calib.html>
- Flow ratio: <https://www.orcaslicer.com/wiki/calibration/flow_ratio_calib.html>
- Retraction: <https://www.orcaslicer.com/wiki/calibration/retraction_calib.html>
- Tolerance: <https://www.orcaslicer.com/wiki/calibration/tolerance_calib.html>
- `filament_shrinkage_compensation_z`: <https://github.com/OrcaSlicer/OrcaSlicer/blob/07a12f5fe46ea450975c4cb65a653f1bb94eaaa4/src/libslic3r/PrintConfig.cpp>
- `Print::shrinkage_compensation`: <https://github.com/OrcaSlicer/OrcaSlicer/blob/07a12f5fe46ea450975c4cb65a653f1bb94eaaa4/src/libslic3r/Print.cpp>
- `CalibMode`: <https://github.com/OrcaSlicer/OrcaSlicer/blob/07a12f5fe46ea450975c4cb65a653f1bb94eaaa4/src/libslic3r/calib.hpp>
- Calibration menu: <https://github.com/OrcaSlicer/OrcaSlicer/blob/07a12f5fe46ea450975c4cb65a653f1bb94eaaa4/src/slic3r/GUI/MainFrame.cpp>

Ellis' Print Tuning Guide (primary for its author's method):

- Tuning order: <https://ellis3dp.com/Print-Tuning-Guide/articles/index_tuning.html>
- Extrusion multiplier: <https://ellis3dp.com/Print-Tuning-Guide/articles/extrusion_multiplier.html>
- Pressure advance: <https://ellis3dp.com/Print-Tuning-Guide/articles/pressure_linear_advance/introduction.html>
- Misconceptions: <https://ellis3dp.com/Print-Tuning-Guide/articles/misconceptions.html>
- Max volumetric flow: <https://ellis3dp.com/Print-Tuning-Guide/articles/determining_max_volumetric_flow_rate.html>
- Cooling: <https://ellis3dp.com/Print-Tuning-Guide/articles/cooling_and_layer_times.html>
- Retraction: <https://ellis3dp.com/Print-Tuning-Guide/articles/retraction.html>

Other:

- CNC Kitchen, "Calibration Cubes are BAD!":
  <https://www.cnckitchen.com/blog/calibration-cubes-are-bad-this-is-how-you-calibrate-your-3d-printer>
- Klipper, Skew correction: <https://www.klipper3d.org/Skew_Correction.html>
- PrusaSlicer `filament_shrinkage_compensation_xy` / `_z` definitions:
  <https://github.com/prusa3d/PrusaSlicer/blob/30ef59195e0f3ee6f270b185bb5f9fb5f349f81f/src/slic3r-shared/src/Slic3r/Biz/Config/Legacy/PrintConfig.cpp>
  and scaling: <https://github.com/prusa3d/PrusaSlicer/blob/30ef59195e0f3ee6f270b185bb5f9fb5f349f81f/src/libslic3r/src/libslic3r/ShrinkageCompensation.cpp>
- PrusaSlicer shipped shrink values (`*shrinkage_ABS*` and siblings):
  <https://github.com/prusa3d/PrusaSlicer/blob/30ef59195e0f3ee6f270b185bb5f9fb5f349f81f/resources/presets/prusa-research-fff/PrusaResearch/preset-filament-common.yaml>
