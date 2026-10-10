# Anatomy of a Bambu Studio filament preset

Research for issue #3 on the filament-lab map (#1). The question: what does a Bambu Studio filament preset contain, and what does a new Filament Profile for an arbitrary Vendor need?

All code facts come from upstream `master` at commit `47a13eeb1` (app version `02.08.04.57`), read without modification. Code is cited by file path and symbol name. Counts were computed by scripts that parse `src/libslic3r/Preset.cpp`, `src/libslic3r/PrintConfig.cpp`, `src/slic3r/GUI/Tab.cpp`, `resources/profiles/BBL.json` and the filament JSON files. None was typed by hand.

## Short answer

- **A filament preset is a flat JSON map with 142 settings keys** (`src/libslic3r/Preset.cpp` `s_Preset_filament_options`), plus `filament_settings_id` and a few header fields. The Filament tab's build code places 112 of them, some only in developer mode. The other 30 are never shown: compatibility lists, circle compensation, drying and internal tuning.
- **For a new Vendor, about 15 keys need deliberate values.** Four set identity: `filament_vendor`, `filament_type`, `compatible_printers`, `inherits`. Eleven are physical values: diameter, density, softening temperature, flow ratio, max volumetric speed, the nozzle temperature range, the nozzle temperatures and the bed temperatures for the plate in use. 106 keys can be copied from a same-type template unchanged.
- **Two settings do nothing on a Bambu printer.** `pressure_advance` and `enable_pressure_advance` are only emitted for non-Bambu printers. On a P1S the pressure-advance K value lives on the printer, keyed by `filament_id`.
- **`filament_id` is the Filament Profile's identity.** The AMS stores it per slot (`tray_info_idx`), Studio maps a slot back to a preset by it, the printer's PA history is filtered by it, and Filament Manager spools point at it. A custom filament gets `"P"` plus 7 hex chars of `MD5("<vendor> <type> <serial>[@<user_id>]")`. System filaments carry Bambu codes such as `GFSNL02`.
- **`setting_id` is the cloud record id of one concrete preset.** System children carry it in their JSON. A user preset gets it from the cloud on first upload and stores it in its `.info` file. `base_id` records the parent's `setting_id`.
- **System presets are three-tier.** A type template (`fdm_filament_pla`) feeds a product `@base` (`instantiation: false`, holds `filament_id` and the vendor), which feeds per-printer children (`instantiation: true`, holds `setting_id` and `compatible_printers`). There is no third-party `@BBL P1S` child: P1S nozzles are listed in the `@BBL X1C` children.
- **There are two kinds of user preset, and only one has its own identity.**
  - "Create Filament" makes *root* presets (`inherits: ""`), one per printer and nozzle. Each stores the full config plus its own `filament_id` under `filament/base/`.
  - "Save as" makes *derived* presets. They store only diffs, always share the parent's `filament_id`, and never appear in the AMS slot list.
- **A custom-vendor root preset shows up in two places.**
  - The AMS slot editor lists it only if the printer reports `support_user_preset`. It is grouped under its Vendor and labelled "Vendor Type Serial".
  - The Filament Manager always lists it, under its Vendor. A spool added for it stores that preset's `filament_id`.
- **The Filament Manager has no local spool file.** `wgtFilaManagerStore::load` and `save` are no-ops, so spools live only in Bambu Cloud and need login. The map's note about `filament_inventory/spools.json` and `resources/web/fila_manager/` is stale.

## 1. What a preset file looks like

### System preset files (`resources/profiles/BBL/filament/`)

A system filament JSON holds header fields, then a sparse set of settings keys that override its parent. For example, `SUNLU/SUNLU PLA Matte @base.json` has `inherits: fdm_filament_pla`, `filament_id: GFSNL02` and `instantiation: false`, plus vendor, cost, density, flow, volumetric speed, temperature range and start G-code. `SUNLU PLA Matte @BBL X1C.json` has `inherits: SUNLU PLA Matte @base`, `setting_id: GFSNLS02`, `instantiation: true`, `include: [fdm_filament_template_direct_dual]`, per-variant arrays and a `compatible_printers` list that includes `Bambu Lab P1S 0.4 nozzle`.

Header fields are read by `PresetBundle::load_vendor_configs_from_json` (its `parse_config` lambda). The JSON key names come from the `BBL_JSON_KEY_*` macros in `src/libslic3r/Preset.hpp`.

| Header field | Meaning |
| --- | --- |
| `type` | `filament` |
| `name` | Preset name; also the key other presets use in `inherits` and `include` |
| `inherits` | Parent preset name; its resolved config is the starting point |
| `include` | Array of template names; only the keys a template changes from the defaults are applied, after the parent and before the file's own keys |
| `from` | `system` for shipped files, `User` for saved user presets |
| `instantiation` | `"true"` creates a selectable preset. `"false"` keeps the config only as a parent or template, so `@base` files never become presets. |
| `filament_id` | Set on `@base`; children inherit it through `inherits` when absent. An instantiated system filament with no resolvable `filament_id` fails to load ("Can not find filament_id"). |
| `setting_id` | Set on each instantiated child; can be refreshed from the cloud by `PresetBundle::update_system_preset_setting_ids` |
| `description`, `alias`, `renamed_from` | Optional. Without an alias, the alias is the name up to `@`, trimmed. |

The vendor index `resources/profiles/BBL.json` lists every file under `filament_list`, with its `sub_path`.

### User preset files

`Preset::save` writes the JSON through `ConfigBase::save_to_json`. The file has header keys `version`, `name` and `from: "User"`, then the settings:

- **Derived preset** (non-empty `inherits`): only the keys that differ from the parent. Per-extruder-variant arrays use `nil` for "same as parent".
- **Root preset** (empty `inherits`, non-empty `filament_id`): the full config, plus `filament_id`.

A user file has no `type` key; its type comes from the folder it sits in. `filament_settings_id` holds the preset's own name.

`Preset::save_info` writes the `.info` sidecar as INI `key = value` lines:

| `.info` key | Meaning |
| --- | --- |
| `sync_info` | Pending cloud action: `create`, `update`, `delete`, or empty. `hold` (skip this session) is never written to disk. |
| `user_id` | Cloud user that created it |
| `setting_id` | Cloud id of this preset; empty until first upload |
| `base_id` | `setting_id` of the parent (derived presets) |
| `updated_time` | UTC seconds; decides which side wins in a cloud sync |

## 2. The settings keys

`s_Preset_filament_options` lists 142 keys. The default filament preset (`PresetBundle::PresetBundle`) is built from them, plus `filament_settings_id`. When a file is loaded, `Preset::remove_invalid_keys` drops any key the default preset lacks. This is why keys such as `filament_contact_safe`, which appear in `fdm_filament_common.json` but are not defined in `PrintConfig.cpp`, are silently discarded.

Three properties cut across the groups below:

- **Per-extruder-variant arrays.** 43 keys are in `filament_options_with_variant` (`src/libslic3r/PrintConfig.cpp`). They hold one value per printer extruder variant. For the P1S (`Bambu Lab P1S 0.4 nozzle.json` `printer_extruder_variant`) that means index 0 = `Direct Drive Standard` and index 1 = `Direct Drive High Flow`, matched through `filament_extruder_variant`. Flow ratio, max volumetric speed, nozzle temperatures and the retraction overrides are all per-variant.
- **Nullable overrides.** The retraction override keys are generated from `filament_extruder_override_keys` (plus `filament_retract_length_nc`). In the default preset they are `nil`, which means "use the printer preset's value". The "Setting Overrides" page checkbox toggles between `nil` and a value.
- **`filament_type` is an open enum with 33 listed values** (`PrintConfig.cpp` `filament_type`). The instantiated system presets use exactly 33 distinct types, the same set: ABS, ABS-GF, ASA, ASA-AERO, ASA-CF, BVOH, EVA, HIPS, PA, PA-CF, PA-GF, PA6-CF, PC, PCTG, PE, PE-CF, PET-CF, PETG, PETG-CF, PHA, PLA, PLA-AERO, PLA-CF, PP, PP-CF, PP-GF, PPA-CF, PPA-GF, PPS, PPS-CF, PVA, TPU, TPU-AMS. The create-filament dialog offers only types that some loaded preset already uses (`CreateFilamentWebDialog::send_init_data`). Variants such as "PLA+" therefore go in the serial, not the type.

### How to read the tables

- **Source** says where a value should come from for a new Vendor:
  - *Identity*: decides which preset this is and where it applies.
  - *Spec sheet*: the vendor's technical data sheet normally gives it.
  - *Measure*: comes from a Calibration Test on the owner's printer.
  - *Template*: safe to copy from a same-type system preset (for example `Generic PLA` or an existing third-party PLA) and leave alone.
  - *No effect on BBL*: ignored on Bambu printers.
- **Min** marks the minimum viable set: keys a new profile must set deliberately rather than inherit.
- **Tab location** is the Filament tab page and group from `TabFilament::build` and `TabFilament::add_filament_overrides_page`; *hidden* means not shown.
- **PrintConfig default** is the built-in default. A root user preset that omits a key falls back to it, not to the type template.

Counts by source: Identity 4, Spec sheet 25, Measure 5, Template 106, No effect on BBL 2 (142 in total). 15 keys are marked Min.

### Identity and compatibility (8)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_vendor` | Identity | yes | Filament / Basic information | `"(Undefined)"` | Brand string. Groups presets in the AMS selector and the Filament Manager. The create dialogs reject `Bambu` and `Generic`. |
| `filament_type` | Identity | yes | Filament / Basic information | `"PLA"` | Must be one of the enum values (see below). Sent to the AMS as `tray_type`, used for the `Generic <type>` fallback, and gates which create-dialog base presets are offered. |
| `compatible_printers` | Identity | yes | hidden | `empty` | Exact printer preset names, for example `Bambu Lab P1S 0.4 nozzle`. The AMS slot editor and slicing compatibility both match on these strings. |
| `compatible_printers_condition` | Template |  | hidden | `empty` | Expression; leave empty. |
| `compatible_prints` | Template |  | hidden | `empty` | Leave empty. |
| `compatible_prints_condition` | Template |  | hidden | `empty` | Leave empty. |
| `inherits` | Identity | yes | hidden | `empty` | Empty for a root profile that owns its `filament_id`; a parent preset name for a derived preset. |
| `filament_extruder_variant` | Template |  | hidden | `"Direct Drive Standard"` | Must line up with the printer's `printer_extruder_variant`; for the P1S that is `Direct Drive Standard`, `Direct Drive High Flow`. Copy from the template. |

### Material properties (16)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_diameter` | Spec sheet | yes | Filament / Basic information | `1.75` | 1.75 for every Bambu printer. |
| `filament_density` | Spec sheet | yes | Filament / Basic information | `0.` | g/cm3; drives weight and cost estimates. |
| `filament_cost` | Spec sheet |  | Filament / Basic information | `0.` | Price per kg; estimates only. |
| `default_filament_colour` | Spec sheet |  | Filament / Basic information | `empty` | Display colour only. |
| `filament_soluble` | Spec sheet |  | Filament / Basic information | `false` |  |
| `filament_is_support` | Spec sheet |  | Filament / Basic information | `false` | Marks a support material. |
| `required_nozzle_HRC` | Spec sheet |  | Filament / Basic information | `0` | Set for abrasive (CF/GF) materials; 0 disables the check. |
| `temperature_vitrification` | Spec sheet | yes | Filament / Basic information | `100` | Labelled "Softening temperature". Feeds the `min_vitrification_temperature` G-code placeholder, the arrange check for mixing plates, and pre-print warnings. Spec sheets give HDT or Vicat. |
| `filament_shrink` | Measure |  | Filament / Basic information | `100` | XY shrink percent applied to slice contours. This is what the planned measured-cube test produces. |
| `filament_velocity_adaptation_factor` | Template |  | Filament / Basic information | `1.0` |  |
| `impact_strength_z` | Template |  | Filament / Basic information | `0` | Developer mode only. |
| `filament_adhesiveness_category` | Template |  | Filament / Basic information | `0` | Developer mode only; prime-tower behaviour. |
| `filament_metal_stickiness` | Template |  | Filament / Basic information | `fmsNone` |  |
| `filament_printable` | Template |  | Filament / Basic information | `3` | Developer mode only; extruder bitmask. |
| `filament_extruder_compatibility` | Template |  | hidden | `0` | Developer mode only; hidden. |
| `filament_notes` | Template |  | Notes / Notes | `empty` | Free text. |

### Flow and volumetric speed (6)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_flow_ratio` | Measure | yes | Filament / Basic information | `1.` | Flow-rate calibration result. Per extruder variant. |
| `filament_max_volumetric_speed` | Measure | yes | Filament / Volumetric speed limitation | `2.` | Volumetric-speed tower result; spec sheets rarely give it. The PrintConfig default is 2 mm3/s, so never leave it unset in a hand-written root file. |
| `filament_adaptive_volumetric_speed` | Template |  | Filament / Volumetric speed limitation | `false` |  |
| `volumetric_speed_coefficients` | Template |  | hidden | `empty` | Hidden; Bambu-fitted polynomial. |
| `enable_pressure_advance` | No effect on BBL |  | Filament / Basic information | `false` | Emitted only for non-Bambu printers (`GCode.cpp` checks `is_BBL_Printer()`). On a P1S the K value lives on the printer, keyed by `filament_id`. |
| `pressure_advance` | No effect on BBL |  | Filament / Basic information | `0.02` | Same as above. |

### Temperatures (15)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `nozzle_temperature_range_low` | Spec sheet | yes | Filament / Basic information | `190` | Sent to the AMS as `nozzle_temp_min`. |
| `nozzle_temperature_range_high` | Spec sheet | yes | Filament / Basic information | `240` | Sent to the AMS as `nozzle_temp_max`. |
| `nozzle_temperature` | Measure | yes | Filament / Print temperature | `200` | Start from the spec sheet; the temp tower refines it. Per extruder variant. |
| `nozzle_temperature_initial_layer` | Spec sheet | yes | Filament / Print temperature | `200` | Per extruder variant. |
| `textured_plate_temp` | Spec sheet | yes | Filament / Print temperature | `45` | The P1S default bed type is Textured PEI Plate. 0 means the filament does not support that plate. |
| `textured_plate_temp_initial_layer` | Spec sheet | yes | Filament / Print temperature | `45` |  |
| `hot_plate_temp` | Spec sheet |  | Filament / Print temperature | `45` | Smooth PEI / High Temp plate. |
| `hot_plate_temp_initial_layer` | Spec sheet |  | Filament / Print temperature | `45` |  |
| `cool_plate_temp` | Spec sheet |  | Filament / Print temperature | `35` |  |
| `cool_plate_temp_initial_layer` | Spec sheet |  | Filament / Print temperature | `35` |  |
| `eng_plate_temp` | Spec sheet |  | Filament / Print temperature | `45` |  |
| `eng_plate_temp_initial_layer` | Spec sheet |  | Filament / Print temperature | `45` |  |
| `supertack_plate_temp` | Spec sheet |  | Filament / Print temperature | `35` |  |
| `supertack_plate_temp_initial_layer` | Spec sheet |  | Filament / Print temperature | `35` |  |
| `chamber_temperatures` | Template |  | Filament / Print temperature | `0` | Only printers with chamber heating act on it. |

### Cooling (26)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `fan_min_speed` | Template |  | Cooling / Part cooling fan | `20` |  |
| `fan_max_speed` | Template |  | Cooling / Part cooling fan | `100` |  |
| `fan_cooling_layer_time` | Template |  | Cooling / Part cooling fan | `60` |  |
| `slow_down_layer_time` | Template |  | Cooling / Part cooling fan | `5` |  |
| `slow_down_min_speed` | Template |  | Cooling / Part cooling fan | `10.` |  |
| `close_fan_the_first_x_layers` | Template |  | Cooling / Part cooling fan | `1` |  |
| `first_x_layer_part_fan_speed` | Template |  | Cooling / Part cooling fan | `0` |  |
| `full_fan_speed_layer` | Template |  | Cooling / Part cooling fan | `0` |  |
| `reduce_fan_stop_start_freq` | Template |  | Cooling / Part cooling fan | `false` |  |
| `slow_down_for_layer_cooling` | Template |  | Cooling / Part cooling fan | `true` |  |
| `no_slow_down_for_cooling_on_outwalls` | Template |  | Cooling / Part cooling fan | `false` |  |
| `cooling_slowdown_logic` | Template |  | Cooling / Part cooling fan | `(int)cslUniformCooling` |  |
| `cooling_perimeter_transition_distance` | Template |  | Cooling / Part cooling fan | `10.0` |  |
| `enable_overhang_bridge_fan` | Template |  | Cooling / Part cooling fan | `true` |  |
| `overhang_fan_speed` | Template |  | Cooling / Part cooling fan | `100` |  |
| `overhang_fan_threshold` | Template |  | Cooling / Part cooling fan | `(int)Overhang_threshold_bridge` |  |
| `overhang_threshold_participating_cooling` | Template |  | Cooling / Part cooling fan | `(int) Overhang_threshold_participating_c` |  |
| `pre_start_fan_time` | Template |  | Cooling / Part cooling fan | `0.0` |  |
| `ironing_fan_speed` | Template |  | Cooling / Part cooling fan | `-1` |  |
| `close_additional_fan_first_x_layers` | Template |  | Cooling / Auxiliary part cooling fan | `1` |  |
| `first_x_layer_fan_speed` | Template |  | Cooling / Auxiliary part cooling fan | `0` |  |
| `additional_fan_full_speed_layer` | Template |  | Cooling / Auxiliary part cooling fan | `0` |  |
| `additional_cooling_fan_speed` | Template |  | Cooling / Auxiliary part cooling fan | `0` |  |
| `activate_air_filtration` | Template |  | Cooling / Exhaust fan | `false` |  |
| `during_print_exhaust_fan_speed` | Template |  | Cooling / Exhaust fan | `60` |  |
| `complete_print_exhaust_fan_speed` | Template |  | Cooling / Exhaust fan | `80` |  |

### Retraction and speed overrides (22)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_retraction_length` | Measure |  | Setting Overrides / Retraction | `nil (printer value)` | Nullable: `nil` means use the printer value. The Calibration tab has a retraction test. |
| `filament_z_hop` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_z_hop_types` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_retraction_speed` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_deretraction_speed` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_retract_length_nc` | Template |  | Setting Overrides / Retraction | `10.` |  |
| `filament_retract_restart_extra` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_retraction_minimum_travel` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_retract_when_changing_layer` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_wipe` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_retract_before_wipe` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_wipe_distance` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_long_retractions_when_cut` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `filament_retraction_distances_when_cut` | Template |  | Setting Overrides / Retraction | `nil (printer value)` |  |
| `override_process_overhang_speed` | Template |  | Setting Overrides / Speed | `false` |  |
| `filament_enable_overhang_speed` | Template |  | Setting Overrides / Speed | `nil (printer value)` |  |
| `filament_bridge_speed` | Template |  | Setting Overrides / Speed | `nil (printer value)` |  |
| `filament_overhang_1_4_speed` | Template |  | Setting Overrides / Speed | `nil (printer value)` |  |
| `filament_overhang_2_4_speed` | Template |  | Setting Overrides / Speed | `nil (printer value)` |  |
| `filament_overhang_3_4_speed` | Template |  | Setting Overrides / Speed | `nil (printer value)` |  |
| `filament_overhang_4_4_speed` | Template |  | Setting Overrides / Speed | `nil (printer value)` |  |
| `filament_overhang_totally_speed` | Template |  | Setting Overrides / Speed | `nil (printer value)` |  |

### Filament change, ramming, prime tower and flushing (23)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_ramming_volumetric_speed` | Template |  | Filament / Volumetric speed limitation | `-1` |  |
| `filament_ramming_volumetric_speed_nc` | Template |  | Filament / Volumetric speed limitation | `-1` |  |
| `filament_ramming_travel_time` | Template |  | Filament / Basic information | `0` |  |
| `filament_ramming_travel_time_nc` | Template |  | Filament / Basic information | `0` |  |
| `filament_pre_cooling_temperature` | Template |  | Filament / Basic information | `0` |  |
| `filament_pre_cooling_temperature_nc` | Template |  | Filament / Basic information | `0` |  |
| `filament_change_length` | Template |  | Filament / Basic information | `10` |  |
| `filament_change_length_nc` | Template |  | Filament / Basic information | `10` |  |
| `filament_prime_volume` | Template |  | Filament / Basic information | `45.` |  |
| `filament_prime_volume_nc` | Template |  | Filament / Basic information | `60.` |  |
| `filament_cooling_before_tower` | Template |  | Filament / Basic information | `10` |  |
| `filament_tower_interface_pre_extrusion_dist` | Template |  | Filament / Basic information | `10.` |  |
| `filament_tower_interface_pre_extrusion_length` | Template |  | Filament / Basic information | `0.` |  |
| `filament_tower_ironing_area` | Template |  | Filament / Basic information | `4.` |  |
| `filament_tower_interface_purge_volume` | Template |  | Filament / Basic information | `20.` |  |
| `filament_tower_interface_print_temp` | Template |  | Filament / Basic information | `-1` |  |
| `filament_minimal_purge_on_wipe_tower` | Template |  | hidden | `15.` |  |
| `filament_preheat_temperature_delta` | Template |  | hidden | `0` |  |
| `filament_flush_volumetric_speed` | Template |  | Multi Filament / Multi Filament | `0` |  |
| `filament_flush_temp` | Template |  | Multi Filament / Multi Filament | `0` |  |
| `filament_flush_temp_fast` | Template |  | Multi Filament / Multi Filament | `0` |  |
| `long_retractions_when_ec` | Template |  | Multi Filament / Multi Filament | `false` |  |
| `retraction_distances_when_ec` | Template |  | Multi Filament / Multi Filament | `10` |  |

### Scarf seam (4)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_scarf_seam_type` | Template |  | Filament / Filament scarf seam settings | `0` |  |
| `filament_scarf_height` | Template |  | Filament / Filament scarf seam settings | `FloatOrPercent(10, true)` |  |
| `filament_scarf_gap` | Template |  | Filament / Filament scarf seam settings | `FloatOrPercent(0, 0)` |  |
| `filament_scarf_length` | Template |  | Filament / Filament scarf seam settings | `10` |  |

### Circle and hole compensation (hidden) (12)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `circle_compensation_speed` | Template |  | hidden | `200` |  |
| `counter_coef_1` | Template |  | hidden | `0` |  |
| `counter_coef_2` | Template |  | hidden | `0.025` |  |
| `counter_coef_3` | Template |  | hidden | `-0.11` |  |
| `hole_coef_1` | Template |  | hidden | `0` |  |
| `hole_coef_2` | Template |  | hidden | `-0.025` |  |
| `hole_coef_3` | Template |  | hidden | `0.28` |  |
| `counter_limit_min` | Template |  | hidden | `-0.04` |  |
| `counter_limit_max` | Template |  | hidden | `0.05` |  |
| `hole_limit_min` | Template |  | hidden | `0.08` |  |
| `hole_limit_max` | Template |  | hidden | `0.25` |  |
| `diameter_limit` | Template |  | hidden | `50` |  |

### Drying (hidden, used by AMS and chamber drying) (8)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_dev_ams_drying_temperature` | Spec sheet |  | hidden | `0` | The PLA template carries four values. |
| `filament_dev_ams_drying_time` | Spec sheet |  | hidden | `0` | The PLA template carries four values. |
| `filament_dev_ams_drying_heat_distortion_temperature` | Spec sheet |  | hidden | `0` |  |
| `filament_dev_drying_softening_temperature` | Spec sheet |  | hidden | `0` |  |
| `filament_dev_drying_cooling_temperature` | Template |  | hidden | `0` |  |
| `filament_dev_chamber_drying_bed_temperature` | Template |  | hidden | `0` |  |
| `filament_dev_chamber_drying_time` | Template |  | hidden | `0` |  |
| `filament_dev_ams_drying_ams_limitations` | Template |  | hidden | `empty` |  |

### Custom G-code (2)

| Key | Source | Min | Tab location | PrintConfig default | Notes |
| --- | --- | --- | --- | --- | --- |
| `filament_start_gcode` | Template |  | Advanced / Filament start G-code | `" "` | The type templates switch fan `P3` by bed temperature; keep the template text. |
| `filament_end_gcode` | Template |  | Advanced / Filament end G-code | `" "` |  |

## 3. Inheritance

### System profile tiers

`resources/profiles/BBL.json` `filament_list` has 2050 entries. 1928 are instantiated (selectable) system presets. 122 are not instantiated: 100 product `@base` files plus the type and extruder templates. 1500 files use `include`.

1. **Type templates.** `fdm_filament_common` is the root. `fdm_filament_<type>` (for example `fdm_filament_pla`) inherits it and sets type-level defaults: cooling, plate temperatures, drying, start G-code.
2. **Extruder templates.** `fdm_filament_template_direct_dual`, `..._direct_bowden` and their `_e3d` variants are pulled in through `include`. They supply the per-variant arrays (for example `filament_extruder_variant = [Direct Drive Standard, Direct Drive High Flow]`).
3. **Product `@base`.** For example `SUNLU PLA Matte @base`. It inherits a type template and holds what is true of the product on any printer: `filament_id`, `filament_vendor`, cost, density, temperature range, softening temperature, product flow and speed. `instantiation: false`, so it is never a selectable preset.
4. **Per-printer child.** For example `SUNLU PLA Matte @BBL X1C`. It inherits `@base`, holds `setting_id`, `compatible_printers` and the printer-specific diffs (nozzle temperature, fan, plate temperatures, per-variant arrays), and is instantiated.

`parse_config` in `PresetBundle::load_vendor_configs_from_json` resolves each file in this order:

1. Start from the parent's resolved config, or the default preset when there is no parent.
2. Apply each `include`, but only the keys the template changes from the defaults.
3. Apply the file's own keys.
4. Inherit `filament_id` from the parent's entry when the file does not set it.

A missing parent fails that file ("Can not find inherits").

### Per-printer children and the P1S

Children are named `<product> @BBL <model>[ <nozzle> nozzle]`. A child applies to exactly the printer preset names in its `compatible_printers`, not to the model in its name. For the owner's printer, 96 instantiated system filaments list `Bambu Lab P1S 0.4 nozzle`:

| Name suffix | Count |
| --- | --- |
| `@BBL X1C` | 73 |
| no suffix (`Generic ABS`, `Generic PLA`, ...) | 10 |
| `@BBL P1S 0.4 nozzle` | 6 |
| `@BBL P1S` | 4 |
| `@BBL X1C 0.4 nozzle` | 2 |
| `@BBL P1P 0.4 nozzle` | 1 |

Every third-party filament usable on a P1S comes from an `@BBL X1C` child: Polymaker 12, SUNLU 7, Overture 2, eSUN 1. The instantiated system set has only 6 vendors in total: Bambu Lab, Generic, Polymaker, SUNLU, Overture and eSUN. An Amazon brand will always be a user preset.

### Compatibility

`is_compatible_with_printer` (`src/libslic3r/Preset.cpp`) decides compatibility:

- A preset is compatible when the active printer preset's name is in `compatible_printers`.
- A preset whose list is empty and whose `compatible_printers_condition` is empty is compatible with everything.
- A system preset is also restricted to printers of its own vendor bundle.
- A user printer preset (a customised P1S) matches through its parent printer (`is_compatible_with_parent_printer`).
- When `PresetCollection::load_presets` loads a root user filament with an empty `compatible_printers`, it fills the list from the text after `@` in the preset name and re-saves the file.

The Filament tab's "Dependencies" page that would edit these lists is compiled out (`#if 0` in `TabFilament::build`), so `compatible_printers` is set only by the create dialogs or by editing the file.

### User presets: derived versus root

| | Derived ("Save as" in the Filament tab) | Root ("Create Filament" dialog) |
| --- | --- | --- |
| Made by | `PresetCollection::save_current_preset` | `PresetCollection::clone_presets_for_filament` (through `clone_presets`) |
| `inherits` | The system or root preset it was saved from | Empty |
| Stored config | Diff against the parent (`Preset::save` with a parent config) | Full config, plus `filament_id` |
| `filament_id` | Always the parent's: `PresetCollection::load_presets` overwrites it from the parent on every load | Its own |
| Path | `<user dir>/filament/<name>.json` | `<user dir>/filament/base/<name>.json` (`PresetCollection::path_for_preset` uses `is_base_preset`) |
| `base_id` | Parent's `setting_id` | Empty |
| In the AMS slot editor | No: only root presets are listed | Yes, when the printer supports user presets |

A user preset cannot inherit from a `@base` file, because `@base` never becomes a preset. Its `inherits` must name an instantiated system preset or a user root preset. If the parent is missing at load time, the user preset is skipped with an error.

## 4. Identity: `filament_id`, `setting_id`, `base_id`

### How they are assigned

| | System preset | User root preset | User derived preset |
| --- | --- | --- | --- |
| `filament_id` | Bambu code set on `@base` (all 100 `@base` ids start `GF`, for example `GFSNL02`), inherited by children | `"P"` + first 7 hex of MD5 of `"<Vendor> <Type> <Serial>"`, with `"@<user_id>"` appended when logged in | Parent's |
| `setting_id` | Per child, in the JSON (all 1928 start `GF`, for example `GFSNLS02` or `GFSL19`) | Issued by the cloud on first upload; empty until then | Same |
| `base_id` | n/a | Empty | Parent's `setting_id` |

The id generator is `make_filament_id` in `src/slic3r/GUI/CreateFilamentWebDialog.cpp`; the legacy `get_filament_id` in `CreatePresetsDialog.cpp` uses the same algorithm.

- If any existing preset already has the same "Vendor Type Serial" name, its `filament_id` is reused. This is how adding a printer to an existing custom filament keeps one identity.
- If the hash collides with a different name, it is re-hashed with a timestamp.

So one Filament Profile = one `filament_id` shared by one root preset per printer and nozzle.

### Why they matter

- **AMS slot mapping.** Setting a slot sends `MachineObject::command_ams_filament_settings` (MQTT `ams_filament_setting`) with:
  - `tray_info_idx = filament_id`
  - `setting_id`
  - `tray_type = filament_type`
  - `nozzle_temp_min` and `nozzle_temp_max` from `nozzle_temperature_range_low` and `nozzle_temperature_range_high`
  - the colour

  The printer reports `tray_info_idx` back. In `DevFilaSystem.cpp` it is parsed into the confusingly named `DevAmsTray::setting_id`, and the MQTT `setting_id` lands in `DevAmsTray::filament_setting_id`.

  When Studio syncs the AMS into the plate's filament list (`PresetBundle::sync_ams_list`), it picks the compatible root preset whose `filament_id` equals the tray's. If none matches, it falls back to a system preset named `Generic <tray type>`, then to the previous selection. A user preset that is not on this machine (not synced, or deleted) therefore degrades to Generic.
- **Printer-side calibration.** `PACalibResult` (`src/libslic3r/Calib.hpp`) carries `filament_id` and `setting_id`. The AMS slot editor filters the printer's PA history by `filament_id` (`AMSTraySettingBase::update_pa_profile_items`). A distinct `filament_id` per Filament Profile is what keeps its K values apart from other filaments' K values.
- **Filament Manager spool link.** `FilamentSpool::filament_id` (`src/slic3r/GUI/fila_manager/wgtFilaManagerStore.h`) is serialised as `setting_id` locally and as `filamentId` in the cloud (`wgtFilaManagerCloudSync.cpp`). Upstream renamed the field in commit `72cb27b99`.
- **Cloud sync.** `setting_id` is the key for `put_setting` and `delete_setting`. The pair (name, `setting_id`) plus `updated_time` decides whether to pull (`PresetCollection::need_sync`). A root preset's upload carries `filament_id`; a derived preset's upload carries `base_id` instead (`PresetCollection::get_differed_values_to_update`).

## 5. Where user presets live and how they sync

### On disk

Paths are relative to `data_dir()`, the app's per-user data directory. They are described from code; no local install was read.

```text
<data_dir>/user/<folder>/filament/<name>.json        derived user presets
<data_dir>/user/<folder>/filament/<name>.info
<data_dir>/user/<folder>/filament/base/<name>.json   root ("custom") user presets
<data_dir>/user/<folder>/filament/base/<name>.info
<data_dir>/system/BBL/filament/...                   cached copy of the vendor bundle
```

`<folder>` is the cloud user id when logged in (`GUI_App` sets `preset_folder` to the user id) and `default` otherwise (`DEFAULT_USER_FOLDER_NAME`). `PresetCollection::load_presets` loads `base/` first, so roots exist before the presets that inherit from them. The preset name is the file name.

Load rules worth knowing for "profiles as plain files":

- `version` must parse as a semver whose major part is not greater than the app's major; otherwise the file is skipped.
- A file that fails JSON parsing is **deleted**, together with its `.info`, by `PresetCollection::load_presets`.
- A root file is applied over the built-in defaults, not over a type template. Keys it omits get the PrintConfig defaults: `filament_max_volumetric_speed` 2, `filament_density` 0, `nozzle_temperature` 200.
- Keys the filament schema does not know are dropped (`Preset::remove_invalid_keys`).

### Cloud sync of user presets

Sync runs only when the user is logged in and the "sync user presets" preference (`app_config` `sync_user_preset`) is on.

- **Pull.** `GUI_App::start_sync_user_preset` calls the network agent's `get_setting_list2`, downloading only entries where `need_sync` is true. Then `GUI_App::reload_settings` runs `PresetBundle::load_user_presets` with the cloud map:
  - It loads roots in a first pass and derived presets in a second.
  - It requires `version`, `setting_id`, `user_id` and `base_id` in each record (`PresetCollection::load_user_preset`).
  - The newer `updated_time` wins. A newer local copy is marked `update`.
- **Push.** The same thread wakes roughly every two seconds and calls `GUI_App::sync_preset` for each user preset with pending work (`PresetCollection::get_user_presets`):
  - `create`, or no `setting_id` yet: `request_setting_id`; the returned id is stored in `.info`.
  - `update`: `put_setting`.
  - `delete`: a queued `delete_setting`.

  The payload is the diff against the parent (derived) or the full config (root), plus `version`, `base_id` or `filament_id`, `updated_time` and `type`.
- **Limits.**
  - A payload over 350 KiB is put on `hold` for the session.
  - When the server answers with code 14, further creates of that type are blocked, and the user sees "newly created user presets can only be used locally".
  - The Bambu wiki states the cloud limits as 20 printer, 100 process and 200 filament presets.
- **Saving in the Filament tab** (`Tab::save_preset`) marks a new preset `create` (stamping `user_id`) and an existing one `update`. Deleting marks it `delete`.

## 6. A custom-vendor preset, end to end

### Creation (`CreateFilamentWebDialog`)

The live entry points both open `CreateFilamentWebDialog` (web UI in `resources/web/filament_create/`):

- the Prepare page's "create filament" action (`Plater::priv::on_create_filament`);
- the Filament Manager's `create_custom` action (`FilamentManagerVM::HandlePreset`), with the Vendor prefilled and locked.

The wx dialog `CreateFilamentPresetDialog` in `src/slic3r/GUI/CreatePresetsDialog.cpp`, which has the "Input Custom Vendor" text box and the "Can't find vendor I want" checkbox, has no callers left. It survives only as the reference the web dialog was ported from. Per-printer additions to an existing custom filament still use `CreatePresetForPrinterDialog` from `EditFilamentWebDialog`.

1. **Vendor, Type, Serial.**
   - Vendor comes from a hard-coded list of 27 brands or from "Add vendor brand". The extra vendor lives only for the dialog session, is capped at 50 characters and is checked for case-insensitive duplicates.
   - `build_and_validate_filament_name` strips `@`, `;` and control characters, trims, and rejects `Bambu` or `Generic`.
   - Type comes from the types already used by loaded presets.
   - The filament name is `"<Vendor> <Type> <Serial>"`. A name that is already an alias prompts before continuing.
2. **Printer and template.** For each selected printer and nozzle, the user picks a base preset: a visible system preset (or a from-scratch user root preset) of that type, compatible with that printer. The parameters shown in the dialog (`send_filament_params`) are read-only. The `create_filament_confirm` payload carries only vendor, type, serial and the printer-to-base-preset pairs.
3. **Clone.** `do_clone_filament_presets` calls `clone_presets_for_filament` once per pair. It copies the template's full config and overrides only `filament_vendor`, `filament_type` and `compatible_printers` (the single printer preset name). It also sets `filament_settings_id`, gives every clone the same new `filament_id`, clears `inherits` and `setting_id`, and saves under `filament/base/` as `"<Vendor> <Type> <Serial> @<printer preset name>"`, for example `Acme PLA Silk @Bambu Lab P1S 0.4 nozzle`. Tuning happens afterwards in the Filament tab, which saves the root preset in place.
4. **Alias.** `PresetCollection::set_custom_preset_alias` sets the alias to the text before `@` ("Acme PLA Silk") unless another preset already holds that alias.

### In the AMS slot editor (`AMSMaterialsSetting`)

`AMSMaterialsSetting::get_filaments_info` builds the list in two passes over root presets:

- **Pass 1:** system presets.
- **Pass 2:** user presets, but only when the printer reports `is_support_user_preset`. `MachineObject` sets it from the firmware's capability bits or its `support_user_preset` field.

In both passes a preset is listed when its `compatible_printers` contains one of the printer preset names for this printer model and slot nozzle (`PresetBundle::get_printer_names_by_printer_type_and_nozzle`) and its `filament_id` is non-empty. Entries are de-duplicated by `filament_id` and labelled by alias.

`FilamentSelectDialog` shows these entries on its "System presets" tab, grouped by `filament_vendor`, so a custom Vendor gets its own group there. Choosing one goes through the filament blacklist check (`DevFilaBlacklist`). It then sends `tray_info_idx` = the custom `filament_id`, `setting_id` = that preset's cloud id (empty if never synced), `tray_type` = `filament_type` and the temperature range from the preset.

The Bambu wiki says custom filaments selectable in the AMS need printer firmware support; it names X1/X1C firmware 1.6.6 or later. It does not say whether a P1S supports it.

The dialog's other tab, "Filament Manager", lists cloud spools grouped by spool brand. A spool is selectable only when `PresetBundle::get_filament_by_filament_id` finds a preset with the spool's `filament_id` that lists this printer; otherwise it is dimmed under "Unsupported Filaments".

### In the Filament Manager

The live Filament Manager is the device web page: `resources/web/device_page/dist`, built from `src/slic3r/GUI/DeviceWeb/device_page/`, driven by `FilamentManagerVM`. The older `wgtFilaManagerPanel` and `resources/web/fila_manager/` are never instantiated.

- **Brand and material lists.** `FilamentManagerVM::build_preset_options` lists every root system preset and every user preset, root or derived, with no printer filter. They are grouped `filament_vendor` → `filament_type` → alias, and each item carries `filament_id`, `setting_id` and an `is_user` flag. A custom Vendor is its own brand. When the Add dialog fills a spool from an AMS tray, its lookup skips `is_user` items, so a tray never resolves to a user preset there.
- **A manually added spool** (`AddEditDialog.tsx`) stores `brand` = Vendor, `series` = alias and `material_type`. Its identity is the cloud catalogue's `filamentId` when (brand, name) matches a Bambu cloud filament setting, otherwise the matched preset's `filament_id`. For a custom Vendor, that means the `P...` id.
- **Storage.** Spools are cloud records only. `wgtFilaManagerStore::load` and `save` are no-ops ("local persistence removed"), `build_spool_list` returns nothing when logged out, and the wiki says changes need an internet connection. `wgtFilaManagerStore::get_storage_path` still names `filament_inventory/spools.json`, but nothing reads or writes it.

## 7. Consequences for the filament-lab design

- **A Filament Profile maps naturally to a set of root user presets sharing one `filament_id`.** One preset per printer model and nozzle, named `"<Vendor> <Type> <Serial> @<printer preset name>"`. That `filament_id` is what the AMS slot, the printer's PA history and Filament Manager spools all key on. A derived ("Save as") preset cannot carry its own identity.
- **Root preset files are already "plain files".** A root preset is a complete, self-contained JSON with a `.info` sidecar. Generating one by hand is possible, but it must contain every key (or be cloned from a template), have a valid `version` and parse cleanly. Omitted keys fall back to PrintConfig defaults, and a malformed file is deleted on load.
- **Not every Calibration Test result belongs in the preset.** Flow ratio, max volumetric speed, nozzle temperature, retraction length and shrink do. Pressure advance does not: on Bambu printers it is printer-side state keyed by `filament_id` (and nozzle), not a preset value.
- **Per-printer values already have a home.** The `@<printer>` naming and `compatible_printers` express "this Filament Profile on this printer and nozzle". The per-extruder-variant arrays also separate Standard from High Flow nozzles. That overlaps with the planned Calibration Record keying (Filament Profile × printer model × nozzle size).

## 8. Open questions this raised

- Does the owner's P1S firmware report `support_user_preset`? If it does not, root user presets never appear in its AMS slot editor, and AMS binding of a custom Filament Profile is impossible through upstream UI.
- Should a Calibration Record be the per-printer root preset itself, or a separate file applied over it at slice time? The preset already carries most measured values per printer and nozzle; PA is the exception.
- How should the fork treat cloud sync for profile files? Root presets sync whole, and the cloud caps filament presets at 200; a Vendor with many products times printers and nozzles could approach that.
- Is it acceptable for the fork to depend on the Filament Manager, given that spools are cloud-only and need login? The map currently assumes a local `spools.json`.

## Sources

Code (upstream `master`, commit `47a13eeb1`):

- `src/libslic3r/Preset.cpp`: `s_Preset_filament_options`, `Preset::save`, `Preset::save_info`, `Preset::load_info`, `Preset::remove_invalid_keys`, `PresetCollection::load_presets`, `PresetCollection::clone_presets`, `PresetCollection::clone_presets_for_filament`, `PresetCollection::save_current_preset`, `PresetCollection::get_differed_values_to_update`, `PresetCollection::load_user_preset`, `PresetCollection::set_sync_info_and_save`, `PresetCollection::need_sync`, `PresetCollection::get_user_presets`, `PresetCollection::get_preset_base`, `PresetCollection::set_custom_preset_alias`, `PresetCollection::get_preset_alias`, `PresetCollection::path_from_name`, `is_compatible_with_printer`
- `src/libslic3r/Preset.hpp`: `BBL_JSON_KEY_*` macros, `PRESET_USER_DIR`, `PresetCollection::is_base_preset`
- `src/libslic3r/PresetBundle.cpp`: `PresetBundle::PresetBundle`, `PresetBundle::load_vendor_configs_from_json` (`parse_config`), `PresetBundle::load_user_presets`, `PresetBundle::import_json_presets`, `PresetBundle::get_filament_by_filament_id`, `PresetBundle::sync_ams_list`, `PresetBundle::update_system_preset_setting_ids`
- `src/libslic3r/PresetBundle.hpp`: `DEFAULT_USER_FOLDER_NAME`
- `src/libslic3r/PrintConfig.cpp`: option definitions (`filament_type`, `filament_vendor`, `filament_max_volumetric_speed`, `pressure_advance`, `temperature_vitrification` and the rest), `filament_extruder_override_keys`, `filament_overhang_override_keys`, `filament_options_with_variant`, `DynamicPrintConfig::get_filament_type`
- `src/libslic3r/GCode.cpp`: pressure-advance emission guarded by `is_BBL_Printer()`; `min_vitrification_temperature` placeholder
- `src/libslic3r/Calib.hpp`: `PACalibResult`
- `src/libslic3r/Config.cpp`: `ConfigBase::save_to_json`
- `src/slic3r/GUI/Tab.cpp`: `TabFilament::build`, `TabFilament::add_filament_overrides_page`, `Tab::save_preset`
- `src/slic3r/GUI/CreateFilamentWebDialog.cpp`: `make_filament_id`, `collect_user_filament_presets`, `send_init_data`, `send_device_info`, `send_filament_params`, `build_and_validate_filament_name`, `do_clone_filament_presets`
- `src/slic3r/GUI/CreatePresetsDialog.cpp`: `get_filament_id`, `CreateFilamentPresetDialog` (legacy), `CreatePresetForPrinterDialog`
- `resources/web/filament_create/step1.js` and `step2.js`: vendor list, "Add vendor brand", `create_filament_confirm` payload
- `src/slic3r/GUI/AMSMaterialsSetting.cpp`: `AMSMaterialsSetting::get_filaments_info`, `sCheckFilamentInfo`, `AMSMaterialsSetting::get_filament_by_id`, `AMSTraySettingBase::update_pa_profile_items`
- `src/slic3r/GUI/FilamentSelectDialog.cpp`: `FilamentSelectDialog::fill_manager_tab`, `FilamentSelectDialog::fill_default_tab`
- `src/slic3r/GUI/DeviceManager.cpp`: `MachineObject::command_ams_filament_settings`, `is_support_user_preset` parsing
- `src/slic3r/GUI/DeviceCore/DevFilaSystem.cpp` and `DevFilaSystem.h`: `DevAmsTray` (`tray_info_idx` → `setting_id`)
- `src/slic3r/GUI/GUI_App.cpp`: `GUI_App::start_sync_user_preset`, `GUI_App::sync_preset`, `GUI_App::reload_settings`
- `src/slic3r/GUI/DeviceWeb/ViewModels/FilamentManager/FilamentManagerVM.cpp`: `FilamentManagerVM::HandlePreset`, `FilamentManagerVM::build_preset_options`, `FilamentManagerVM::build_spool_list`
- `src/slic3r/GUI/DeviceWeb/device_page/src/features/filament-manager/AddEditDialog.tsx`: spool identity on save
- `src/slic3r/GUI/fila_manager/wgtFilaManagerStore.h` and `.cpp`: `FilamentSpool`, `wgtFilaManagerStore::load`, `wgtFilaManagerStore::save`; `wgtFilaManagerCloudSync.cpp`: cloud field mapping
- `resources/profiles/BBL.json`, `resources/profiles/BBL/filament/` (`fdm_filament_common.json`, `fdm_filament_pla.json`, `fdm_filament_template_direct_dual.json`, `SUNLU/SUNLU PLA Matte @base.json`, `SUNLU/SUNLU PLA Matte @BBL X1C.json`, `SUNLU/SUNLU PLA Matte @BBL P1P.json`), `resources/profiles/BBL/machine/Bambu Lab P1S 0.4 nozzle.json`, `resources/profiles/BBL/machine/Bambu Lab P1S.json`

Bambu Lab wiki:

- [Creating custom filaments in Bambu Studio](https://wiki.bambulab.com/en/bambu-studio/create-filament): the two creation methods, the "Vendor Type Serial @Printer" naming, AMS support from X1/X1C firmware 1.6.6, and the cloud limits of 20 printer, 100 process and 200 filament presets.
- [How to Use the Filament Manager in Bambu Studio](https://wiki.bambulab.com/en/software/bambu-studio/filament-manager): spools stored in Bambu Cloud, internet required, manual add for third-party spools.
