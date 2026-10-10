# How upstream calibration stores and applies results

Research for issue #4 on the wayfinder map (#1). Read against upstream
`master` at commit `47a13eeb1d9a` (2026-10-08). Paths are relative to the
repository root. Code is cited by file and symbol, not by line number.
Glossary terms (Filament Profile, Spool, AMS Slot, Calibration Test,
Calibration Record) follow `filament-lab/CONTEXT.md` on the `filament-lab`
branch.

## Short answer

| Test (`CalibMode`) | What the owner picks, and the value | Where it is stored | Keyed by | When it reaches a print |
| --- | --- | --- | --- | --- |
| PA line / PA pattern, Calibration tab, manual (`Calib_PA_Line`, `Calib_PA_Pattern`) | Best line or sharpest corner, giving a K value plus a name | Printer-side PA table (K history), via MQTT `extrusion_cali_set`. On P1-series printers whose firmware reports no `cali_version`, the K goes straight onto one AMS Slot instead | The printer the table lives on, then `filament_id`, nozzle diameter and nozzle flow type. Multi-extruder printers add `extruder_id`; nozzle-rack printers add nozzle position and serial. `setting_id` and the name are stored but not used to look entries up | On the printer at print time, through the K profile bound to the AMS Slot (`cali_idx`). The owner binds it by hand in the Device tab. Nothing goes into the sliced file or the print command |
| Auto PA (`Calib_Auto_PA_Line`) | Nothing to pick: the printer measures K (and `n_coef`) for each selected filament | Same printer-side PA table | Same as above | Same as above. Supported printers also offer a per-print "Flow Dynamics Calibration" option. The P1 series does not |
| PA line / pattern / tower from the Calibration menu (`Calib_PA_Tower` exists only here) | Best line, corner or tower height, giving K | Nowhere. The owner types K into Manage Result > New, or into the AMS Slot dialog | (as above, once entered) | As above, once entered |
| Flow rate (`Calib_Flow_Rate`) | Smoothest top surface. Coarse pass, then optional fine pass, gives `filament_flow_ratio` | Wizard: a new or overwritten user filament preset (local JSON, synced to the Bambu cloud when logged in). Menu: nowhere | The preset. It inherits `filament_id` and printer compatibility from its parent. No printer serial, and no nozzle unless the parent preset is nozzle-specific | At slice time, automatically, but only when that preset is the one selected for the filament in Prepare |
| Temperature tower (`Calib_Temp_Tower`) | Best block, giving a nozzle temperature | Nowhere | None | At slice time, once the owner edits `nozzle_temperature` in a filament preset |
| Volumetric-speed tower (`Calib_Vol_speed_Tower`) | Height where the wall fails, giving mm³/s | Menu: nowhere. A wizard that would save `filament_max_volumetric_speed` exists but is never shown | None | At slice time, once the owner edits the preset |
| VFA tower (`Calib_VFA_Tower`) | Speed band with the fewest vertical artifacts, in mm/s | Nowhere | None | At slice time, once the owner edits the process preset's `outer_wall_speed` |
| Retraction tower (`Calib_Retraction_tower`) | Height where stringing stops, giving a length in mm | Nowhere | None | At slice time, once the owner edits `filament_retraction_length` (filament override) or `retraction_length` (printer) |

For the owner's P1S with a 0.4 mm nozzle:

- PA calibration is manual only.
- The PA result is saved to the printer's table. Studio refuses to add a
  table entry once a nozzle diameter already holds 16 entries, counted across
  all filaments (`MAX_PA_HISTORY_RESULTS_NUMS`).
- PA reaches a print only through the AMS Slot binding.
- Flow rate is manual only and lands in a filament preset.
- No other test is saved anywhere.

Upstream never applies PA through a preset on a Bambu printer. The filament
preset's `pressure_advance` is hidden and never emitted for Bambu printers.

## Two entry points

Upstream has two separate ways to run a Calibration Test, and they differ in
whether anything gets saved.

1. **Calibration tab (device-connected wizards).**
   `src/slic3r/GUI/CalibrationPanel.cpp` `CalibrationPanel::init_tabpanel`
   creates only `PressureAdvanceWizard` ("Flow Dynamics") and
   `FlowRateWizard`. The `MaxVolumetricSpeedWizard` line is commented out, and
   `CALI_MODE_COUNT` in `src/slic3r/GUI/CalibrationPanel.hpp` is 2. These
   wizards send the test to the printer themselves and end with a save step.
   Implemented in `src/slic3r/GUI/CalibrationWizard.cpp` and
   `src/slic3r/Utils/CalibUtils.cpp`.
2. **Calibration menu in Prepare (the "OrcaSlicer calibrations" block).**
   `src/slic3r/GUI/MainFrame.cpp` adds Temperature, Flow rate (Coarse/Fine),
   Pressure advance, Retraction test, and More > Max flowrate / VFA. These
   call `Plater::calib_pa`, `Plater::calib_flowrate`, `Plater::calib_temp`,
   `Plater::calib_max_vol_speed`, `Plater::calib_retraction` and
   `Plater::calib_VFA` in `src/slic3r/GUI/Plater.cpp`. The dialogs are in
   `src/slic3r/GUI/calib_dlg.cpp`. Each one starts a new project and loads a
   test model. Most of them also **edit the currently selected print,
   filament and printer presets in place**, which leaves those presets marked
   modified. Each then calls `Print::set_calib_params`, so that the G-code
   generator varies a parameter along the test (by height in
   `GCode::process_layer` for the towers). The owner slices and sends the plate like any other print.
   **There is no save step.** The owner reads the result off the print and
   enters it wherever it belongs.

`CalibUtils::calib_temptue`, `CalibUtils::calib_VFA` and
`CalibUtils::calib_retraction` exist but have no callers.
`CalibUtils::calib_max_vol_speed` is called only from the hidden
`MaxVolumetricSpeedWizard`.

## Printer-family branches

The client decides the family with `MachineObject::get_printer_series` in
`src/slic3r/GUI/DeviceManager.cpp`. That function maps `series_x1` and
`series_o` to `SERIES_X1`, and everything else, including the A1's
`series_n`, to `SERIES_P1P`. Several code paths also check
`PrinterArch::ARCH_I3`, which sends the A1 family down the X1 path. The P1S
(`resources/printers/C12.json`) is `series_p1p` / `core_xy`.

| Behaviour | X1 family | A1 family (`ARCH_I3`) | P1P / P1S |
| --- | --- | --- | --- |
| Auto PA button (`CalibrationPAStartPage::update`) | Shown when the printer reports PA support | Shown when the printer reports PA support | Hidden; manual only |
| Auto flow-rate button (`CalibrationFlowRateStartPage::update`) | Shown when `is_support_flow_calibration` | Hidden (falls in the `SERIES_P1P` branch) | Hidden; manual only |
| PA save path (`PressureAdvanceWizard::on_cali_save`) | `CalibUtils::set_PA_calib_result` → `extrusion_cali_set` with a full entry | Same as X1 | If `DevCalib::IsVersionInited` (firmware reports `cali_version`): same full entry. Otherwise legacy per-slot `MachineObject::command_extrusion_cali_set` (K only, `n_coef` fixed at 1.4) |
| PA table cap enforced by the client | None | 16 per nozzle diameter | 16 per nozzle diameter |
| Per-print "Flow Dynamics Calibration" option (`SelectMachineDialog`, `flow_cali`) | Shown when the printer reports `is_support_pa_calibration` | Same | Never: `MachineObject` forces `is_support_pa_calibration` to false for `series_p1p` |

The Bambu wiki agrees: "P1 supports only manual calibration", and the
pre-print option "P1 series printers do not have this option as they do not
support automatic calibration".

## Per test

### PA line and PA pattern from the Calibration tab (manual)

- **Run.** `PressureAdvanceWizard::on_cali_start` with
  `CalibrationMethod::CALI_METHOD_MANUAL` builds a `CalibInfo` from the AMS
  Slot, filament preset, nozzle and plate. It then calls
  `CalibUtils::calib_generic_PA`, which slices
  `resources/calib/pressure_advance/pressure_advance_test.stl` (line) or
  `pa_pattern.3mf` (pattern) with a start/end/step range and sends the job.
  On the P1 series, the client first warns when the PA table for that nozzle
  diameter already holds `MAX_PA_HISTORY_RESULTS_NUMS` entries.
- **Pick.** `CaliPASaveManualPanel::get_result` takes a K (validated against
  `MIN_PA_K_VALUE` and `MAX_PA_K_VALUE` in `CalibUtils.cpp`) and a name. It
  copies `tray_id`, `nozzle_diameter`, `filament_id`, `setting_id`,
  `extruder_id`, `nozzle_pos_id`, `nozzle_sn` and `nozzle_volume_type` from
  the cached selection (`DevCalib::GetSelectedCalibPreset`).
- **Save.** `PressureAdvanceWizard::can_save_cali_result` asks before
  overwriting an entry with the same name and `filament_id` (and the same
  extruder and flow type on multi-extruder printers). Outside the X1 family it
  refuses to save once the table holds 16 entries for that nozzle diameter.
  `MachineObject::command_set_pa_calibration` then publishes
  `extrusion_cali_set` with these fields for each filament: `tray_id`,
  `cali_idx` (if editing), `extruder_id`, `nozzle_id` (for example `HS00-0.4`,
  built by `_generate_nozzle_id` from flow type and diameter),
  `nozzle_diameter`, `ams_id`, `slot_id`, `filament_id`, `setting_id`,
  `name`, `k_value`, `n_coef` (`"0.0"` for manual), and on nozzle-rack
  printers `nozzle_pos` and `nozzle_sn`.
- **Legacy P1 path.** When the firmware reports no `cali_version`,
  `CaliPASaveP1PPanel::get_result` takes only K. The client then sends
  `command_extrusion_cali_set(tray_id, …)`, whose payload carries only
  `tray_id`, `k_value`, fixed `n_coef` 1.4 and temperatures. That stores K on
  the AMS Slot itself, with no history entry.

### Auto PA (`Calib_Auto_PA_Line`)

- Printers without `DevCalib::IsSupportNewAutoCali` (`CALI_METHOD_AUTO`,
  taken only on the X1 family or `ARCH_I3`): `CalibUtils::calib_PA` →
  `MachineObject::command_start_pa_calibration` publishes `extrusion_cali`
  with one `filaments[]` entry per selected AMS Slot (`filament_id`,
  `setting_id`, `nozzle_id`, `nozzle_diameter`, temperatures,
  `max_volumetric_speed`). The printer prints and measures.
  `DevCalib::ExtrusionCalibGetResultParse` reads the results, and the owner
  can rename or edit them before `set_PA_calib_result(…, true)` saves them
  with `n_coef`.
- Printers where `DevCalib::IsSupportNewAutoCali` holds:
  `CalibUtils::calib_generic_auto_pa_cali` slices
  `resources/calib/pressure_advance/auto_pa_line_single.3mf` or
  `auto_pa_line_dual.3mf` and sends it as a print job. Results come back and
  are saved the same way.
- Not available on the P1 series.

### PA line, pattern and tower from the Calibration menu

`Plater::calib_pa` handles `Calib_PA_Line`, `Calib_PA_Pattern` and
`Calib_PA_Tower`. The tower is `_calib_pa_tower`, and `GCode::process_layer`
sets K = `start + int(z) * step` with `GCodeWriter::set_pressure_advance`
(`M900 K…`). The line and pattern tests also write an `M900` for each line
or pattern, from `CalibPressureAdvanceLine` / `CalibPressureAdvancePattern` in
`src/libslic3r/calib.cpp`. Nothing is saved. To use the result the owner must
add it to the printer table, or type K into the AMS Slot dialog. Adding it
uses "New" under Manage Result (`NewCalibrationHistoryDialog` in
`src/slic3r/GUI/CaliHistoryDialog.cpp`), which sends `extrusion_cali_set`
with `tray_id` -1.

### Flow rate (`Calib_Flow_Rate`)

- **Manual (all printers).** `FlowRateWizard::on_cali_start` →
  `CalibUtils::calib_flowrate(pass, …)` slices
  `resources/calib/filament_flow/flowrate-test-pass1.3mf` or
  `flowrate-test-pass2.3mf`. Each block sets a per-object
  `print_flow_ratio` from the modifier encoded in its object name.
  `CalibrationFlowCoarseSavePage` and `CalibrationFlowFineSavePage` compute
  the new value as `base_ratio * (100 + modifier) / 100`. `base_ratio` is the
  ratio the pass was printed around, stashed in `DevCalib`. For a coarse
  pass that is the selected preset's `filament_flow_ratio`
  (`CalibrationPresetPage::get_cali_stage`). For a fine pass it is the coarse
  result (`FlowRateWizard::cache_coarse_info`) or the ratio the owner typed.
- **Auto (X1 family only).** `CalibUtils::calib_flowrate_X1C` →
  `MachineObject::command_start_flow_ratio_calibration` publishes
  `flowrate_cali`. The printer returns a ratio, read by
  `DevCalib::FlowrateGetResultParse`. The client code has no flow-ratio table
  on the printer.
- **Save.** `FlowRateWizard::on_cali_save` writes `filament_flow_ratio` into a
  filament preset through `CalibrationWizard::save_preset` (auto) or
  `CalibrationWizard::save_preset_with_index` (manual, which writes the one
  array index chosen by `get_index_for_extruder_parameter` for that extruder
  and nozzle flow type). The default name is "`<filament>` Flow Rate
  Calibrated" (`get_default_name` in `CalibrationWizardSavePage.cpp`). If the
  owner reuses the source preset's name, the save overwrites that user preset.
  It cannot overwrite a system preset. The new preset is a copy of the
  selected one. It inherits from that preset, or from that preset's parent
  when the selected one was itself derived
  (`PresetCollection::save_current_preset`). Either way it keeps the
  `filament_id` and printer compatibility it was copied with. Its `setting_id` is cleared and assigned later by cloud
  sync, because `save_preset` sets `sync_info` to `create` or `update`. It is
  written as JSON under the user preset folder
  (`PresetCollection::path_from_name`).
- **Menu path.** `Plater::calib_flowrate` prints the same models. Nothing is
  saved; the owner works out the ratio and edits the preset.

### Temperature tower (`Calib_Temp_Tower`)

`Plater::calib_temp` loads
`resources/calib/temperature_tower/temperature_tower.stl` and sets the start
temperature. `GCode::process_layer` then lowers the temperature by 5 °C every
10 mm (`set_temperature(start - offset)`). Nothing is saved. The owner sets
`nozzle_temperature` and `nozzle_temperature_initial_layer` in a filament
preset.

### Volumetric-speed tower (`Calib_Vol_speed_Tower`)

`Plater::calib_max_vol_speed` prints `SpeedTestStructure.step` in vase mode.
It raises `filament_max_volumetric_speed` to 200 for the test, converts the
mm³/s range to mm/s, and `GCode::process_layer` sets `outer_wall_speed` =
`start + z * step`. Nothing is saved from the menu. The hidden
`MaxVolumetricSpeedWizard::on_cali_save` would write
`filament_max_volumetric_speed` (0 to 60) to a new preset named
"… Max Vol Speed Calibrated". The tab is commented out, so the owner cannot
reach that save.

### VFA tower (`Calib_VFA_Tower`)

`Plater::calib_VFA` prints `resources/calib/vfa/VFA.stl` in vase mode, and
`GCode::process_layer` steps `outer_wall_speed` every 5 mm. The result is a
speed (or a band to avoid) for the process preset, not the filament. Nothing
is saved.

### Retraction tower (`Calib_Retraction_tower`)

`Plater::calib_retraction` prints
`resources/calib/retraction/retraction_tower.stl`. `GCode::process_layer`
applies `retraction_length = start + floor(max(0, z - 0.4)) * step` to the
writer. Nothing is saved. The natural home is the filament override
`filament_retraction_length` (listed in `filament_extruder_override_keys` in
`src/libslic3r/PrintConfig.cpp`) or the printer's `retraction_length`.

## The printer-side PA table in detail

- **Lives on the printer.** The client reads it over MQTT with
  `extrusion_cali_get` (`MachineObject::command_get_pa_calibration_tab`) into
  `DevCalib::m_pa_calib_tab`. Fetches are filtered by `filament_id`, and
  optionally by `nozzle_diameter`, `extruder_id` and `nozzle_id`.
  `DevCalib::PrepareFetchQueue` refetches when the printer's `cali_version`
  changes, one request per nozzle diameter the model supports (or one full
  fetch on multi-extruder and nozzle-rack printers). The client keeps no cloud
  copy, and this code does not show whether the Bambu cloud mirrors the
  table.
- **Entry shape.** `PACalibResult` in `src/libslic3r/calib.hpp` holds
  `cali_idx` (assigned by the printer), `filament_id`, `setting_id`, `name`,
  `k_value`, `n_coef`, `nozzle_diameter`, `nozzle_volume_type` (parsed back
  from `nozzle_id`), `extruder_id`, `nozzle_pos_id`, `nozzle_sn`, `tray_id`,
  `ams_id`, `slot_id` and `confidence`.
- **Lookup key.** `AMSTraySettingBase::update_pa_profile_items` in
  `src/slic3r/GUI/AMSMaterialsSetting.cpp` builds an AMS Slot's PA dropdown
  with a `PaHistoryFilter` (in `src/slic3r/GUI/DeviceCore/DevCalib.h`) on the
  Slot's `tray_info_idx` (the `filament_id`), the nozzle diameter, the nozzle
  flow type when the printer reports one, and `extruder_id` on multi-extruder
  printers. The wizard's duplicate-name check uses name plus `filament_id`
  and nozzle diameter (plus extruder and flow type on multi-extruder
  printers). The P1-series cap counts by nozzle diameter alone. Nozzle
  material (stainless or hardened steel) is not part of any key: `nozzle_id`
  (`_generate_nozzle_id`) encodes only flow type and diameter.
- **Default.** `cali_idx` -1 means "Default", the firmware's built-in value.
  When the printer lacks automatic flow calibration, the client shows a
  default K from `get_default_k_n_value` in `CalibUtils.cpp`: 0.02 for most
  `filament_id`s, 0.04 for a listed set, and higher for TPU.
- **Binding to an AMS Slot.** Choosing a profile in the Device tab's AMS Slot
  dialog calls `AMSTraySettingBase::save_pa_profile_selection` →
  `CalibUtils::select_PA_calib_result` →
  `MachineObject::commnad_select_pa_calibration`, which publishes
  `extrusion_cali_sel` with `tray_id`, `ams_id`, `slot_id`, `cali_idx`,
  `filament_id` and `nozzle_diameter`. The printer echoes the Slot's
  `cali_idx`, and `DevCalib::ExtrusionCalibSelectParse` stores it on
  `DevAmsTray::cali_idx`. Pressing OK in that dialog always sends the
  dropdown's current choice. When the dialog opens on the Slot's current
  filament, it preselects the Slot's current `cali_idx`. When the owner picks
  a different filament, it preselects the PA entry whose name equals the
  chosen filament preset's alias, or Default if none does
  (`AMSMaterialsSetting`, lambda `get_cali_index`). The Reset button sends
  Default (`AMSTraySettingBase::reset_calibration`). The wiki describes
  binding as a manual step: "you can set the Factor K value you want to use
  for each slot in the material dialog on the device page." The client code
  does not show whether `extrusion_cali_set` with a `tray_id` also binds the
  new entry to that Slot.
- **Applied at print time, by the printer.** `PrintParams` in
  `src/slic3r/Utils/bambu_networking.hpp` carries `ams_mapping`,
  `ams_mapping_info` (with `filamentId` per filament), `task_flow_cali`,
  `auto_flow_cali` and `extruder_cali_manual_mode`, but no K or `cali_idx`.
  The printer uses the K bound to whichever AMS Slot each filament maps to.
  `SelectMachineDialog` only warns (`_check_kval_not_default`) when the
  per-print auto calibration would override a non-default Slot binding.

## Why PA never travels in a preset on Bambu printers

- `GCode::set_extruder` and `WipeTowerIntegration::append_tcr` in
  `src/libslic3r/GCode.cpp` emit `set_pressure_advance` only when
  `!is_BBL_Printer() && enable_pressure_advance`.
- `TabFilament::toggle_options` in `src/slic3r/GUI/Tab.cpp` hides
  `enable_pressure_advance` and `pressure_advance` for Bambu printers ("hide
  these useless option for bambu printer").
- The P1S start G-code
  (`resources/profiles/BBL/machine/Bambu Lab P1S 0.4 nozzle template machine_start_gcode.json`)
  contains no `M900`. No BBL filament preset carries `M900` in
  `filament_start_gcode`.
- The firmware does accept `M900 K` from G-code: the PA line, pattern and
  tower tests depend on it, and the X1 start G-code uses it.

## How preset-based results reach a print

`filament_flow_ratio`, `filament_max_volumetric_speed`, temperatures and
retraction all apply at slice time from whichever filament preset is selected
for each filament in Prepare. Selection is the catch. "Sync from AMS"
(`PresetBundle::sync_ams_list` in `src/libslic3r/PresetBundle.cpp`) picks, for
each AMS Slot, the first compatible preset that is a **base** preset
(`get_preset_base(f) == &f`, meaning a system preset or a user root preset)
whose `filament_id` matches the Slot's `tray_info_idx`. A preset that the
flow-rate wizard derived from a system preset inherits from it, so it is not
a base preset and sync never picks it. The owner must select it by hand. The
one exception: if the owner saves the result over a user root preset under
that preset's own name, the root itself carries the new value and sync picks
it.

Filament presets made with Create Filament get a filament id of `P` plus seven
hex digits (`get_filament_id` in `src/slic3r/GUI/CreatePresetsDialog.cpp`).
The id is shared by every printer variant of that filament name. Both the PA
table and AMS sync would key a custom Vendor's filament on that id.

## Other state that is not a result

- `AppConfig::save_printer_cali_infos`, called from
  `CalibrationWizard::back_preset_info`, persists the in-progress wizard
  selection per printer `dev_id`: chosen presets, stashed flow ratio, and
  whether the run finished. This lets the wizard resume after a restart. It
  does not store results.
- Every test sends a telemetry event (`agent->track_event("cali", …)` or
  `"third_cali"`) with the calibration type and `filament_id`. That is not
  storage the slicer reads back.
- `ExtrusionCalibration` (`src/slic3r/GUI/ExtrusionCalibration.cpp`, opened
  from `StatusPanel::on_filament_extrusion_cali`) is an older per-Slot
  dialog. It writes K straight to one Slot with `command_extrusion_cali_set`.

## Implications for the map

- The destination says a Calibration Record is "applied automatically at
  slice time". That holds for flow ratio, max volumetric speed, temperature
  and retraction, once the right preset is selected. It does not hold for PA,
  which upstream applies only on the printer, per AMS Slot.
- Upstream's PA key (printer, `filament_id`, nozzle diameter, nozzle flow
  type) lines up with
  "Filament Profile × printer × nozzle size" if a Filament Profile maps to one
  `filament_id`. The table is per physical printer rather than per printer
  model, and on the P1S it holds at most 16 entries per nozzle diameter across
  all filaments.
- Upstream's flow-rate save produces a separate derived preset, which AMS sync
  will not pick. A record that "applies automatically" needs either in-place
  updates of a root preset or a hook in preset selection.

## Open questions this raised

- Can a Filament Profile carry PA as data, by putting `M900 K…` in its
  `filament_start_gcode`? Would the P1S firmware keep that value, or reapply
  the AMS Slot's K after the filament loads? Only a test print can settle
  this.
- Does the owner's P1S firmware report `cali_version`? That decides between
  the table path and the legacy per-Slot path. It shows in the Calibration tab
  as a "Manage result" button on the Flow Dynamics start page.
- Does `extrusion_cali_set` with a `tray_id` bind the new entry to that AMS
  Slot on the P1S, or must the owner always pick it in the Slot dialog?
- Should a Calibration Record also hold process-side values (VFA's
  `outer_wall_speed`)? The glossary says a Calibration Test produces
  Filament Profile values, and VFA does not.

## Sources

- Code (upstream `master` at `47a13eeb1d9a`):
  - `src/libslic3r/calib.hpp`, `src/libslic3r/calib.cpp`,
    `src/libslic3r/GCode.cpp`, `src/libslic3r/GCodeWriter.cpp`,
    `src/libslic3r/PresetBundle.cpp`, `src/libslic3r/Preset.cpp`,
    `src/libslic3r/PrintConfig.cpp`.
  - `src/slic3r/GUI/CalibrationPanel.cpp`,
    `src/slic3r/GUI/CalibrationWizard.cpp`,
    `src/slic3r/GUI/CalibrationWizardStartPage.cpp`,
    `src/slic3r/GUI/CalibrationWizardSavePage.cpp`,
    `src/slic3r/GUI/CaliHistoryDialog.cpp`, `src/slic3r/GUI/calib_dlg.cpp`,
    `src/slic3r/GUI/ExtrusionCalibration.cpp`, `src/slic3r/GUI/Plater.cpp`,
    `src/slic3r/GUI/MainFrame.cpp`, `src/slic3r/GUI/Tab.cpp`.
  - `src/slic3r/GUI/AMSMaterialsSetting.cpp`,
    `src/slic3r/GUI/SelectMachine.cpp`, `src/slic3r/GUI/Jobs/PrintJob.cpp`,
    `src/slic3r/GUI/DeviceManager.cpp`,
    `src/slic3r/GUI/DeviceCore/DevCalib.h`,
    `src/slic3r/GUI/DeviceCore/DevCalib.cpp`,
    `src/slic3r/GUI/DeviceCore/DevFilaSystem.h`,
    `src/slic3r/GUI/CreatePresetsDialog.cpp`.
  - `src/slic3r/Utils/CalibUtils.hpp`, `src/slic3r/Utils/CalibUtils.cpp`,
    `src/slic3r/Utils/bambu_networking.hpp`.
  - `resources/printers/C12.json` (P1S capability flags) and the other
    `resources/printers/*.json`; `resources/profiles/BBL/machine` and
    `resources/profiles/BBL/filament`.
- Bambu Lab wiki, Flow Dynamics Calibration:
  <https://wiki.bambulab.com/en/software/bambu-studio/calibration_pa>
- Bambu Lab wiki, Flow Rate Calibration:
  <https://wiki.bambulab.com/en/software/bambu-studio/calibration_flow_rate>
