# Upstream hook points and churn for an isolated module

Research for ticket #6 on the filament-lab map (#1).

**Question.** Where can an isolated module hook into upstream Bambu Studio with
the fewest, smallest edits, and how often do those upstream files change?

**Snapshot.** `upstream/master` at `47a13eeb1` (2026-10-08). `master` has the same
tree. All counts below come from `git log` against `upstream/master`.

## Answer in brief

The module needs **one build line**. It can then work with **zero to three call
sites** in upstream code, depending on where Calibration Records live:

1. **Build.** Add one line, `add_subdirectory(GUI/filament_lab)`, to
   `src/slic3r/CMakeLists.txt`. This is the `fila_manager` pattern.
2. **UI entry.** Add one call (plus one `#include`) in `MainFrame::MainFrame`,
   straight after `init_menubar_as_editor()`. From there the module adds its own
   menu items, and optionally a tab, through accessors that are already public.
3. **Slice-time overlay.** Add one call (plus one `#include`) in
   `BackgroundSlicingProcess::apply`. Upstream last touched that function on
   2022-12-15. This hook is needed **only** if Calibration Records are not plain
   user presets.

Preset load and save, the Filament Manager store, and calibration result save
need **no hook**. Public APIs and on-disk files already cover them. All seven
candidate one-line hooks, written against the 2025-09-15 upstream, rebase
cleanly onto today's upstream (see [Method](#method) and the combined probe in §3).

The pure-data route is a **user filament preset JSON file** per Filament Profile
× printer × nozzle, using upstream's own `<filament> @<printer> <nozzle>` naming.
That route needs no C++ and no slice hook at all.

## Method

- **File churn.** For each file, `git log upstream/master --since=2025-10-10
  --oneline -- <file> | wc -l` (committer date). Over the same window,
  `upstream/master` has 3374 commits in total. Bambu lands commits in large
  batches, so file-level counts are high across the board.
- **Symbol churn.** `git log upstream/master --since=2025-10-10 -s -L
  ':<function regex>:<file>'` counts commits that touched the body of the
  function holding the hook. A rebase conflicts only when upstream edits the
  lines next to ours, so this is the sharper proxy.
- **Rebase probe.** With git plumbing and a private index, each one-line hook
  was inserted into the file as it stood at `4f0f0f3e9` (2025-09-15). That is
  the newest upstream commit before the 12-month window. Each probe commit was
  then merged with `git merge-tree --write-tree --merge-base=4f0f0f3e9
  upstream/master <probe>`. Exit 0 means it would have rebased cleanly today.
  Sanity check: a probe that edits a line upstream changed in
  `BackgroundSlicingProcess.cpp` does report a conflict. The probe tests
  textual merging only, not that the hook still compiles.

## 1. The `fila_manager` module pattern

- **Build wiring.** `src/slic3r/GUI/fila_manager/CMakeLists.txt` defines no
  target. It does `list(APPEND SLIC3R_GUI_SOURCES GUI/fila_manager/...)`, then
  `set(SLIC3R_GUI_SOURCES ${SLIC3R_GUI_SOURCES} PARENT_SCOPE)`.
  `src/slic3r/CMakeLists.txt` pulls it in with `add_subdirectory(GUI/fila_manager)`,
  next to `GUI/DeviceCore`, `GUI/DeviceTab` and `GUI/DeviceWeb`, ahead of the
  platform-specific source lists and
  `add_library(libslic3r_gui STATIC ${SLIC3R_GUI_SOURCES})`. The module's files
  compile into `libslic3r_gui` and include each other as `fila_manager/...` or
  `slic3r/GUI/fila_manager/...`. A `filament_lab` directory can copy this
  exactly.
- **Tests.** `tests/fila_manager/CMakeLists.txt` builds a slim test executable.
  It compiles `wgtFilaManagerStore.cpp` directly against `libslic3r`, without
  `libslic3r_gui`. `tests/CMakeLists.txt` adds it with one
  `add_subdirectory(fila_manager)` (2 commits in 12 months). The target's
  comment says the store is wx-free, but that has rotted. Since upstream
  `2f0544dab` (2026-07-14), `wgtFilaManagerStore.cpp` includes `GUI_App.hpp`
  and calls `wxGetApp()` inside `FilamentSpool::to_json`, so the slim target
  probably no longer links. `SLIC3R_BUILD_TESTS` defaults to `OFF` in the
  top-level `CMakeLists.txt`, so nothing notices. (Not verified by a build.)
  If the module keeps its core logic wx-free (record lookup, overlay
  computation) and builds its test target in its own CI, it gets a cheap test
  target for one more line and the guarantee stays enforced.
- **Feature flag.** `wgtFilaManagerFeature.h` `FilaManagerEnabledConfigKey`
  (`studio_enable_fila_manager`) together with
  `is_fila_manager_disabled_by_config` is a pattern worth copying: an
  app-config key, read once, gates construction.
- **What not to copy.** `fila_manager` is not isolated. `GUI_App.hpp` holds a
  raw pointer and an accessor for each of its objects (`GUI_App::fila_manager_store`
  and the others), and `GUI_App::on_init_inner` builds them inline. Upstream
  files outside the module include its headers: `AMSMaterialsSetting.cpp`/`.hpp`,
  `FilamentSelectDialog.hpp`, `GUI_App.cpp`/`.hpp`, `Monitor.cpp`,
  `Preferences.cpp`, `StatusPanel.cpp`/`.hpp` and `FilamentManagerVM.cpp`. The
  fork module should own its singletons and initialise lazily, so `GUI_App` is
  never touched.
- **Dead code.** `wgtFilaManagerPanel` (the legacy panel for
  `resources/web/fila_manager/`) is still compiled, but nothing constructs it
  outside its own file. `FilamentManagerVM` replaced it; its data builders
  carry the comment "migrated from wgtFilaManagerPanel". Do not build on the
  legacy panel.

## 2. Web UI to C++ bridge

Upstream ships three web-UI patterns.

| Pattern | Example | Bridge | Build step | Fit for the module |
|---|---|---|---|---|
| Stand-alone dialog with its own WebView | `CreateFilamentWebDialog`, `EditFilamentWebDialog`; pages in `resources/web/filament_create/` | JS calls `SendWXMessage(json)` (`resources/web/include/globalapi.js`, which calls `window.wx.postMessage`). C++ binds `wxEVT_WEBVIEW_SCRIPT_MESSAGE_RECEIVED` and dispatches on `command`. C++ pushes back with `WebView::RunScript(browser, "HandleStudio(<json>)")`. | None: plain HTML/JS loaded from `file://<resources>/web/...` | **Best.** The pages live in a module-owned `resources/web/filament_lab/`. The whole `resources/` tree is installed or symlinked by the top-level CMake `install(DIRECTORY ...)` and the `src/CMakeLists.txt` symlink step, so a new folder needs no CMake edit. |
| React SPA in the shared device WebView | `DeviceWebHost` / `DeviceWebPage`, SPA source in `src/slic3r/GUI/DeviceWeb/device_page/` | `{module, submod, action, payload}` routed by `DeviceWebManager::Dispatch` to an `IViewModel`. `FilamentManagerVM` is registered in `DeviceWebHost::EnsureBuilt`. | Node + pnpm, downloaded and run by `src/slic3r/GUI/DeviceWeb/CMakeLists.txt` `prepare_node_env`. `resources/web/device_page/dist` is generated, not tracked. | **Avoid.** A new route means editing upstream SPA files (`routes/`, `__root.tsx`, the generated `routeTree.gen.ts`) plus `DeviceWebHost::EnsureBuilt`. That function had 9 commits since the file appeared on 2026-05-30, and the rebase probe of a VM registration line **conflicts**. The SPA README says to register in `WebDevicePage.cpp`, but registration actually happens in `DeviceWebHost::EnsureBuilt`. |
| Legacy panel | `wgtFilaManagerPanel` + `resources/web/fila_manager/` | Same JSON bridge | None | Dead code (see §1). |

**Can new UI be mostly HTML/JS?** Yes. Copy the dialog pattern into the
module: a `DPIDialog` or `wxPanel` that calls `WebView::CreateWebView(this, url)`,
plus a small `command` dispatcher. The module could reuse the
`IViewModel::MakeResponse` wire shape for consistency without registering with
upstream's manager. `CreateFilamentWebDialog.cpp` is 1266 lines, but most of
that is preset-query logic, not bridge code. The module can also open
upstream's own `CreateFilamentWebDialog` and `EditFilamentWebDialog` directly
(public constructors; `CreateFilamentWebDialog::created_filament` returns the
result). `FilamentManagerVM` and `Plater` already do this. Note that
`resources/web/filament_create/` is one month old and moving fast (9 commits
since 2026-09-08), so call that dialog rather than forking its pages.

## 3. Candidate hook points

The two churn columns are commits in the 12-month window. **File** counts
commits touching the file. **Symbol** counts commits touching the function or
region that holds the hook, via `git log -L`. **Probe** is the rebase-probe
result.

| Hook | Upstream file and symbol | Smallest plausible edit | File churn | Symbol churn | Probe |
|---|---|---|---|---|---|
| Build | `src/slic3r/CMakeLists.txt`, the `add_subdirectory(GUI/...)` block before `add_library(libslic3r_gui ...)` | 1 line: `add_subdirectory(GUI/filament_lab)` | 58 | 1 (the commit that added `fila_manager`/`DeviceWeb`) | clean (both a year-old line before `add_library` and a line after `add_subdirectory(GUI/fila_manager)` from 2026-04-28) |
| Tests (optional) | `tests/CMakeLists.txt` | 1 line: `add_subdirectory(filament_lab)` | 2 | n/a | not probed |
| Menu entry | `src/slic3r/GUI/MainFrame.cpp` `MainFrame::MainFrame`, right after the `init_menubar_as_editor()` call | 1 call + 1 `#include`: `filament_lab::install_ui(*this)`. The module appends items to `topbar()->GetCalibMenu()` on Windows and Linux (`BBLTopbar::GetCalibMenu`, public). On macOS it uses the Calibration menu found via `wxFrame::GetMenuBar()`, because `MainFrame::m_calib_menu` is private. | 68 | 0 for the lines around the call; 10 for all of `MainFrame::init_menubar_as_editor`; 1 each for the Windows and macOS Calibration-menu blocks | clean |
| Tab entry (optional) | same call site; `MainFrame::m_tabpanel` is public | 0 extra upstream lines: `install_ui` calls `m_tabpanel->AddPage(...)` | 68 | 16 for `MainFrame::init_tabpanel`; 2 for the Calibration `AddPage` block | clean (a line after the Calibration `AddPage`, from 2026-04-28) |
| App init (alternative to the menu hook) | `src/slic3r/GUI/GUI_App.cpp` `GUI_App::on_init_inner` after `mainframe = new MainFrame()` | 1 call. It needs a second call in `GUI_App::recreate_GUI`, which also does `new MainFrame()` (on a language change, for example). The `MainFrame::MainFrame` site covers both cases, so prefer it. | 100 | 22 for `on_init_inner`; 4 for the Filament Manager init block | clean |
| Slice-time overlay | `src/slic3r/GUI/BackgroundSlicingProcess.cpp` `BackgroundSlicingProcess::apply`, after `new_config.apply(*m_current_plate->config());` | 1 call + 1 `#include`: `filament_lab::overlay_config(new_config, ...)`. Every GUI slice goes through this function: `Plater::priv::update_background_process`, `Plater::apply_background_progress`, `Plater::select_plate` and `Plater::select_plate_by_hover_id` all call `background_process.apply`. The other GUI `Print::apply` callers either load G-code for preview (`PartPlate::load_gcode_from_file`) or build Calibration-tab prints (`CalibUtils::process_and_store_3mf`), so those prints bypass the overlay. The config already carries `filament_settings_id` (the preset per slot), `printer_model`, `printer_settings_id` and `nozzle_diameter`. `Print::calib_params().mode` lets the overlay skip calibration prints (`CalibMode::Calib_None` means a normal print). | 17 | **0** (last touched 2022-12-15) | clean |
| Overlay, rejected alternatives | `src/libslic3r/PresetBundle.cpp` `PresetBundle::full_fff_config`; `src/slic3r/GUI/Plater.cpp` `Plater::priv::update_background_process` | 1 line each | 76 / 309 | 7 / 3 | clean / clean |
| Preset load | `src/libslic3r/PresetBundle.cpp` `PresetBundle::load_user_presets`, `PresetBundle::load_presets` | **None.** The module calls the public `PresetCollection::load_user_preset(name, values, ...)` or writes JSON files (see §4). | 76 | 0 / 1 | n/a |
| Preset save notification | `src/libslic3r/Preset.cpp` `PresetCollection::save_current_preset`; `Preset::save`; `src/slic3r/GUI/Tab.cpp` `Tab::save_preset` | **None preferred.** Read the user preset files on demand. If a notification is ever needed, `PresetCollection::save_current_preset` is the quietest place. | 81 / 81 / 138 | 0 (last 2024-10-28) / 1 / 3 | not probed |
| Filament Manager store | `src/slic3r/GUI/fila_manager/wgtFilaManagerStore.h` `wgtFilaManagerStore`, `FilamentSpool` | **None.** Read through `wxGetApp().fila_manager_store()` (`get_spool`, `find_by_slot`, `find_by_tag_uid`). `FilamentSpool::filament_id` (serialised as `setting_id`) already points a Spool at a filament preset. Keep any extra per-Spool data in a module-owned file keyed by `spool_id`. Adding a field instead touches `FilamentSpool::to_json`, `FilamentSpool::from_json`, `wgtFilaManagerStore::apply_patch` and the cloud mapping. | 13 (file created 2026-04-28) | 6 (`to_json`) / 5 (`from_json`) / 3 (`apply_patch`) | clean (one field in `FilamentSpool::to_json` from 2026-04-28) |
| Calibration result save | `src/slic3r/GUI/CalibrationWizard.cpp` `CalibrationWizard::save_preset`, `CalibrationWizard::save_preset_with_index` | **None needed.** Flow-rate and max-volumetric-speed results already land in a user filament preset through these functions, which call `PresetCollection::save_current_preset`. Pressure-advance results go to the printer through `CalibUtils::set_PA_calib_result`; P1-series uses that path too, in `PressureAdvanceWizard::on_cali_save`. If a notification is wanted: 1 call after `new_preset->save_info()` in each of the two functions. | 20 | **0** each (last 2023-11-01 and 2025-03-22); the `on_cali_save` callers have 3 to 4 | clean |
| PA result on the printer | `src/slic3r/Utils/CalibUtils.cpp` `CalibUtils::set_PA_calib_result` | 1 call, only if PA history must mirror into records | 53 | 0 (last 2023-11-01) | not probed |

**Combined probe.** All seven one-line hooks together, written against the
2025-09-15 base (build, overlay, `MainFrame` call, `GUI_App` call, calibration
save, `full_fff_config`, `update_background_process`), merge cleanly into
today's `upstream/master`. So
do `#include` lines added after either the first or the last `#include` of
`BackgroundSlicingProcess.cpp`, `MainFrame.cpp`, `GUI_App.cpp` and
`CalibrationWizard.cpp`.

**Avoid.** Edits inside `Plater.cpp` (309 commits), `Tab.cpp` (138),
`GUI_App.cpp`/`.hpp` (100/30), the DeviceWeb SPA and `DeviceWebHost` (the probe
conflicts), and new fields in `FilamentSpool`. The `FilamentSpool` field merges
textually, but it has to spread across several functions plus the cloud
schema.

## 4. Pure-data routes (no C++)

- **User filament presets.** These are JSON files in
  `<data_dir>/user/<user id or "default">/filament/` with an optional `.info`
  sidecar. `PresetBundle::load_user_presets` and `PresetCollection::load_presets`
  load them. A file can `inherits` a system preset and override only the keys
  that changed. The values the Calibration Tests produce map to filament keys:
  `nozzle_temperature`, `filament_flow_ratio`, `enable_pressure_advance` +
  `pressure_advance`, `filament_max_volumetric_speed`, `filament_shrink`, and
  `filament_retraction_length` (one of `filament_extruder_override_keys`).
  Upstream already names per-printer filament presets
  `<filament> @<printer> <nozzle>`; for example
  `resources/profiles/BBL/filament/` holds `Bambu ABS @BBL P1S 0.4 nozzle.json`.
  So **a Calibration Record can simply be a user preset**, picked in the plate's
  filament slot like any other preset. That needs no slice hook at all. Hazards
  in `PresetCollection::load_presets`:
  - a file whose JSON fails to parse is **deleted**, along with its `.info`;
  - a file without a parseable `version`, or whose `inherits` parent is
    missing, is skipped with only a log line.

  So keep the source of truth elsewhere, for example in the module's own
  folder or the `filament-lab` branch, and generate the preset files after
  validating them. User presets also take part in Bambu cloud sync when signed
  in (`Preset::sync_info`), which ties into the map's open "Cloud sync" item.
- **Vendor bundle in `<data_dir>/system/`.** `PresetBundle::load_system_presets_from_json`
  loads every `*.json` in that folder. `PresetUpdater::priv::check_installed_vendor_profiles`
  removes only vendors that also ship in `resources/profiles/`. So a
  `FilamentLab.json` + `FilamentLab/` bundle dropped there should load with no
  C++. **This is untested.** Limits from reading
  `PresetBundle::load_vendor_configs_from_json`: `inherits` resolves only inside
  the same bundle (the `configs` map in that function is local), so the bundle
  must carry full base configs; and system presets are read-only in the UI.
  Shipping the bundle in the repo's `resources/profiles/` instead means it is
  copied only when it is `BBL` or enabled in the app-config vendor list.
- **Spool → Filament Profile link.** `FilamentSpool::filament_id` already does
  this through the cloud store. No code is needed to *read* it.

## 5. Findings that change map assumptions

- **The Filament Manager store is no longer a local file.**
  `wgtFilaManagerStore::load` and `wgtFilaManagerStore::save` are no-ops ("local
  persistence removed — store starts empty, populated by cloud pull after
  login"). The change is upstream commit `571efedb2` (2026-07-14).
  `wgtFilaManagerStore::get_storage_path` (`filament_inventory/spools.json`) is
  now unused. Spool writes from the Filament Manager need sign-in and network:
  `FilamentManagerVM` returns "cloud sync requires sign-in and network". The
  Filament Manager is also off by default on macOS
  (`is_fila_manager_disabled_by_config`). The map's note "local store
  `filament_inventory/spools.json`" is out of date.
- **An invisible overlay can change calibration prints.** The menu calibrations
  (`Plater::calib_flowrate`, `Plater::calib_pa` and the others) slice through
  the normal `BackgroundSlicingProcess::apply` path. An overlay there would
  rewrite the very values the test is varying unless it checks
  `Print::calib_params().mode`. The overlay is also invisible in the settings
  tabs, and the project file stores un-overlaid values: `Plater::export_3mf`
  writes `PresetBundle::full_config_secure`, not the sliced config.
- **The UI entry and the build line are the only hooks needed in every design.**
  The overlay hook exists only if records are not presets.

## Questions for the map

- Is a Calibration Record a generated user filament preset
  (`<Filament Profile> @<printer> <nozzle>`, inheriting the profile's base
  preset), or a separate file overlaid at slice time? The answer decides
  whether the `BackgroundSlicingProcess::apply` hook exists at all.
- Filament Manager Spools are cloud-only (sign-in and network). Should the
  module depend on a Bambu account for Spool data, or keep its own local Spool
  file keyed to the cloud `spool_id` when one exists?
- If an overlay is chosen, does it skip every `CalibMode` print, and where does
  the owner see the overlaid values, given that the settings tabs and the
  saved 3MF will not show them?
- Where does the module's UI live: a dialog opened from the Calibration menu
  (no tab-index risk), or a new main tab appended to `MainFrame::m_tabpanel`?
- Does the fork's rebase routine build the module's test target in CI? Upstream's
  own slim `fila_manager` test target appears to have rotted unnoticed.

## Sources

All files are at `upstream/master` `47a13eeb1`.

- `src/slic3r/CMakeLists.txt` (the `add_subdirectory(GUI/...)` block and
  `add_library(libslic3r_gui ...)`)
- `src/slic3r/GUI/fila_manager/CMakeLists.txt`
- `tests/CMakeLists.txt` and `tests/fila_manager/CMakeLists.txt`; top-level
  `CMakeLists.txt` option `SLIC3R_BUILD_TESTS` and the resources
  `install(DIRECTORY ...)`
- `src/CMakeLists.txt` (the resources symlink step)
- `src/slic3r/GUI/fila_manager/wgtFilaManagerFeature.h`, `FilaManagerEnabledConfigKey`
- `src/slic3r/GUI/fila_manager/wgtFilaManagerFeature.cpp`, `is_fila_manager_disabled_by_config`
- `src/slic3r/GUI/fila_manager/wgtFilaManagerStore.h` / `.cpp`: `FilamentSpool`,
  `FilamentSpool::to_json`, `wgtFilaManagerStore::load`, `::save`,
  `::apply_patch`, `::get_storage_path`
- `src/slic3r/GUI/fila_manager/wgtFilaManagerPanel.cpp`
- `src/slic3r/GUI/GUI_App.hpp` / `.cpp`: `GUI_App::on_init_inner`,
  `GUI_App::recreate_GUI`, `GUI_App::fila_manager_store`
- `src/slic3r/GUI/MainFrame.hpp` / `.cpp`: `MainFrame::MainFrame`,
  `MainFrame::init_tabpanel`, `MainFrame::init_menubar_as_editor`,
  `MainFrame::topbar`, `MainFrame::m_tabpanel`, `MainFrame::TabPosition`,
  `MainFrame::show_calibration_button`
- `src/slic3r/GUI/BBLTopbar.hpp`, `BBLTopbar::GetCalibMenu`
- `src/slic3r/GUI/CreateFilamentWebDialog.hpp` / `.cpp`:
  `CreateFilamentWebDialog::OnScriptMessage`, `CreateFilamentWebDialog::run_script`
- `src/slic3r/GUI/EditFilamentWebDialog.hpp`
- `resources/web/include/globalapi.js`, `SendWXMessage`
- `src/slic3r/GUI/Widgets/WebView.cpp`, `WebView::CreateWebView` and `WebView::RunScript`
- `src/slic3r/GUI/DeviceWeb/IViewModel.hpp`
- `src/slic3r/GUI/DeviceWeb/DeviceWebManager.hpp`
- `src/slic3r/GUI/DeviceWeb/DeviceWebHost.cpp`: `DeviceWebHost::EnsureBuilt`,
  `DeviceWebHost::BuildUrl`
- `src/slic3r/GUI/DeviceWeb/CMakeLists.txt`, `prepare_node_env`
- `src/slic3r/GUI/DeviceWeb/device_page/README.md`
- `src/slic3r/GUI/DeviceWeb/ViewModels/FilamentManager/FilamentManagerVM.cpp`
- `src/slic3r/GUI/BackgroundSlicingProcess.cpp`, `BackgroundSlicingProcess::apply`
- `src/slic3r/GUI/Plater.cpp`: `Plater::priv::update_background_process`,
  `Plater::apply_background_progress`, `Plater::select_plate`,
  `Plater::select_plate_by_hover_id`, `Plater::calib_flowrate`, `Plater::export_3mf`
- `src/slic3r/GUI/PartPlate.cpp`, `PartPlate::load_gcode_from_file`
- `src/slic3r/Utils/CalibUtils.cpp`, `CalibUtils::process_and_store_3mf`
- `src/libslic3r/calib.hpp`: `CalibMode`, `Calib_Params`
- `src/libslic3r/Print.hpp`, `Print::calib_params`
- `src/libslic3r/PresetBundle.cpp`: `PresetBundle::full_fff_config`,
  `PresetBundle::full_config_secure`, `PresetBundle::load_presets`,
  `PresetBundle::load_user_presets`, `PresetBundle::load_system_presets_from_json`,
  `PresetBundle::load_vendor_configs_from_json`
- `src/libslic3r/Preset.cpp`: `PresetCollection::load_presets`,
  `PresetCollection::save_current_preset`, `PresetCollection::load_user_preset`,
  `PresetCollection::clone_presets_for_filament`, `Preset::save`
- `src/slic3r/Utils/PresetUpdater.cpp`, `PresetUpdater::priv::check_installed_vendor_profiles`
- `src/slic3r/GUI/CalibrationWizard.cpp`: `CalibrationWizard::save_preset`,
  `CalibrationWizard::save_preset_with_index`, `FlowRateWizard::on_cali_save`,
  `PressureAdvanceWizard::on_cali_save`, `MaxVolumetricSpeedWizard::on_cali_save`
- `src/slic3r/Utils/CalibUtils.cpp`, `CalibUtils::set_PA_calib_result`
- `src/libslic3r/PrintConfig.cpp`, the definitions of `filament_flow_ratio`,
  `enable_pressure_advance`, `pressure_advance`, `filament_max_volumetric_speed`,
  `filament_shrink` and `nozzle_temperature`, and `filament_extruder_override_keys`
- Upstream commits `571efedb2`, `2f0544dab`, `ef2096efd`, `c3949b1a1` and `4f0f0f3e9`
