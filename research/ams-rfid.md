<!-- markdownlint-disable MD013 -->

# AMS RFID and open custom-spool identification

Research for ticket #2 on map #1. Researched 2026-10-10 against upstream `master`
(`47a13eeb1`) and the primary sources listed at the end.

**Confidence tags.** **[High]**: read directly in source code, a spec, or a Bambu Lab
release note or store page. **[Medium]**: a reverse-engineering repo's own docs, or an
inference from code. **[Low]**: forum or community anecdote.

## Short answer

- A Bambu tag is a MIFARE Classic 1K tag. Its sector keys are derived from the tag UID
  with a published HKDF. Its data is signed with RSA-2048, and the AMS rejects any tag
  whose signature does not verify. Nobody outside Bambu can author a new tag the AMS
  trusts. The only way to get one accepted is to clone a genuine Bambu tag
  byte for byte, and the AMS then reports that Bambu product. **[High]** for the key
  derivation script; **[Medium]** for the signature and AMS behaviour, which come from
  reverse-engineering docs that no other source contradicts.
- No AMS model reads any open tag format (OpenSpool, OpenTag3D, TigerTag, OpenPrintTag).
  An unreadable or unverified tag is treated as no tag. The slot becomes a third-party
  slot, which is configured over MQTT with `ams_filament_setting` or on the printer
  screen. **[High]** for "no open format is read"; **[Medium]** for "treated as no tag".
- Every open project examined here that sets AMS slots does it the same way. An external NFC reader
  identifies the spool, and the device then publishes `ams_filament_setting` to the
  printer over LAN MQTT. **[High]**
- Since P1 firmware 01.08.02.00 (2025-06-03), a P1S rejects control commands from
  unauthorized software unless it is in LAN Mode with Developer Mode on. Every
  ESP32 reader option (SpoolEase, OpenSpool, TigerSpool) therefore costs the owner
  cloud features. Bambu Studio sends through Bambu's own network plugin, so a feature in
  this fork that calls `MachineObject::command_ams_filament_settings` should not hit
  this gate. **[High]** for the firmware rule; **[Medium]** for the fork inference.
- Upstream already ships a manual binding of a Filament Manager spool to an AMS slot.
  The binding lives in Bambu Studio and the Bambu cloud. The printer only ever receives
  the filament settings, never a spool identity. **[High]**
- A P1S can use the AMS, AMS 2 Pro and AMS HT; the AMS Lite does not fit P-series
  printers. Every model the owner might own ignores non-Bambu tags, so the unknown AMS
  model does not change this decision. **[High]** for compatibility; **[Medium]** for
  identical behaviour, because no source covers the AMS 2 Pro or AMS HT specifically.

## 1. The Bambu tag

### Hardware

- MIFARE Classic 1K, 13.56 MHz. The guide's recovery script targets the Fudan
  FM11RF08S chip. **[Medium]** (RFID-Tag-Guide README)
- The reverse-engineered AMS RFID board in the guide uses a Fudan FM17580 reader IC
  (`rfid-board` BOM). That board is the first-generation AMS board. The guide documents
  no board for the AMS 2 Pro or AMS HT. **[Medium]**

### Data layout

Per `BambuLabRfid.md` in the RFID-Tag-Guide (**[Medium]**: reverse-engineered, with
some fields marked unconfirmed):

| Block | Content |
| --- | --- |
| 0 | Tag UID (4 bytes) and manufacturer data, read-only |
| 1 | Material variant ID (for example `A00-K0`) and material ID (`GFA00`), 8 chars each |
| 2 | Filament type (`PLA`) |
| 4 | Detailed type (`PLA Basic`) |
| 5 | Colour RGBA, spool weight in grams (uint16 LE), diameter (float) |
| 6 | Drying temperature and hours, bed temperature type, bed temperature, hotend max and min |
| 8 | X-Cam info, nozzle diameter (unconfirmed) |
| 9 | Tray UID, presumably what MQTT reports as `tray_uuid` |
| 10 | Spool width, mm x 100 |
| 12, 13 | Production date and time, ASCII |
| 14 | Filament length in metres (unconfirmed) |
| 16 | Extra colour info for multi-colour filament |
| Each sector trailer | Keys and access bits: always `87 87 87 69`, Key B all zeros |
| Sectors 10 to 15 | RSA-2048 signature |

### Keys

- `deriveKeys.py` `kdf` derives the keys with HKDF-SHA256. The input keying material is
  the tag UID, the salt is a published 16-byte constant, and the context is
  `RFID-A\0`. It outputs 16 six-byte Key A values, one for each sector of a 1K tag.
  **[High]**
- The guide describes the blocks as "encrypted". On MIFARE Classic, the Crypto1 cipher
  protects sector access and the radio link, and the stored data is readable once a
  sector is authenticated. With the key derivation published (the guide dates it
  2024-11-19), any Bambu tag can be read with a Proxmark3, a Flipper Zero or a
  PN532-class reader. **[Medium]**

### Signature: can a new tag be authored that the AMS trusts?

- **No.** "Bambu printers check the content of the tag and then check if the signature is
  valid. If the signature is invalid, it rejects the tag." The signature covers all the
  tag data, and only Bambu holds the private key. **[Medium]** (RFID-Tag-Guide README,
  "How do RFID tags work?")
- **Cloning works.** A clone needs the same UID, the same data blocks and the same
  signature. It therefore carries the identity of the original Bambu product and colour.
  Results by magic-tag generation:
  - Gen 1: rejected. The AMS probes for the Gen 1 unlock command `0x40`.
  - Gen 2 (CUID): sources disagree. The Bambu-Research-Group README still says "Works",
    while the newer `docs/WriteTags.md` in the co-author's fork says "Inconsistent or no
    longer works (AMS writes a winky face to block 0, 'bricking' the tag)". That note was
    added on 2025-06-20, which suggests an AMS firmware countermeasure by mid-2025.
  - Gen 4 FUID and sealed UFUID: work.

  **[Medium]**
- Custom AMS firmware that skips the signature check is only a stated goal in the guide,
  and no source found describes a working one. **[Medium]**
- Dumps of genuine tags for most Bambu products are collected in
  `queengooborg/Bambu-Lab-RFID-Library` (GPL-3.0). It is the input for the cloning
  route. **[High]**

## 2. What the printer reports, and what Bambu Studio does with it

All **[High]**, from the upstream code:

- `DevFilaSystemParser::ParseAmsTrayInfo` (`src/slic3r/GUI/DeviceCore/DevFilaSystem.cpp`)
  reads these fields for each AMS Slot (the code calls it a tray): `tag_uid`,
  `tray_uuid`, `tray_info_idx` (the filament ID, such as `GFA00` or a user preset's ID),
  `tray_type`, `tray_sub_brands`,
  `tray_color`, `cols`, `ctype`, `tray_weight`, `tray_diameter`, `nozzle_temp_min/max`,
  `remain`, `remain_g`, `k`, `n`, `cali_idx`, `state` and `setting_id`. The AMS-level
  `tray_exist_bits`, `tray_reading_bits` and `tray_read_done_bits` give physical
  presence and RFID read progress.
- `DevFilaSystem::IsBBL_Filament` treats a slot as holding a Bambu Spool when `tag_uid`
  is non-zero. A Bambu spool's slot is read-only in the UI. Every other slot is
  "third brand" and editable (`AMSMaterialsSetting::m_is_third`).
- `MachineObject::command_ams_user_settings` sends `ams_user_setting` with
  `startup_read_option` (read tags at power-on), `tray_read_option` (read tags on
  insertion) and `calibrate_remain_flag` (estimate remaining filament from RFID).
- `MachineObject::command_ams_refresh_rfid` (G-code `M620 R`) and
  `MachineObject::command_ams_refresh_rfid2` (command `ams_get_rfid` with `ams_id` and
  `slot_id`) force a slot to re-read its tag.
- The Filament Manager auto-registers and displays `tray_uuid` as a spool's RFID
  identity only when `tag_uid` is 16 hex characters with `01` at offset 12
  (`normalize_ams_rfid_for_web`,
  `wgtFilaManagerSync::check_and_register_new_rfid_spools`). An all-zero `tray_uuid`
  means "no RFID" (`FilamentSpool::is_valid_tag_uid`).

## 3. AMS models and unknown or non-Bambu tags

| Model | Works with a P1S | Reads Bambu RFID | Non-Bambu or unknown tag |
| --- | --- | --- | --- |
| AMS (gen 1) | Yes | Yes. Reads on insertion and at power-on, each user-toggleable | Treated as no tag. The slot is third-party and needs manual settings |
| AMS 2 Pro | Yes, from P1 firmware 01.08.00.00 | Yes. RFID also selects official drying presets | Same as the AMS (assumed: no model-specific source) |
| AMS HT | Yes, from P1 firmware 01.08.00.00 | Yes, through its RFID coil. RFID also selects drying presets | Same as the AMS (assumed: no model-specific source) |
| AMS Lite | **No.** "The AMS lite is not compatible with X, P and H2 series printers" | Yes, per the Bambu store page | Not applicable to the owner |

- Model compatibility and RFID use: Bambu store pages and the P1 release notes.
  **[High]** The "Non-Bambu or unknown tag" column is **[Medium]**: it rests on the
  RFID-Tag-Guide's statement that the printer rejects unsigned tags. A P1 printer takes up to four AMS units in any mix of AMS, AMS 2 Pro and
  AMS HT. **[High]**
- No release note in the P1 series history (latest 01.10.00.00, 2026-03-30) mentions
  reading third-party tags. 01.10.00.00 fixed drying of "non-official filaments" in the
  AMS 2 Pro and AMS HT, so third-party spools are expected in those units, just not
  identified. **[High]**
- What a slot shows after a non-RFID spool goes in: a forum reply (October 2025) says
  the AMS "defaults to the filament previously in that slot". SpoolEase's slot logic
  (`BambuPrinter::get_updated_tray`) notes that the printer remembers colour and K for
  an emptied slot without reporting them. **[Low]**
- Upstream commit `6ba8089c2` skips Filament Manager auto-registration for the AMS Lite
  and says "AMS Lite has no RFID". The Bambu store page says the AMS Lite reads Bambu
  RFID. This contradiction does not matter for a P1S. **[Medium]**

## 4. `ams_filament_setting`: the command that sets a slot

`MachineObject::command_ams_filament_settings` (`src/slic3r/GUI/DeviceManager.cpp`)
publishes this to the printer's request topic **[High]**:

```json
{"print": {
  "command": "ams_filament_setting",
  "sequence_id": "<n>",
  "ams_id": 0, "slot_id": 2, "tray_id": 2,
  "tray_info_idx": "<filament_id, e.g. GFL99 or a user preset id>",
  "setting_id": "<preset setting_id, may be empty>",
  "tray_color": "RRGGBBAA",
  "nozzle_temp_min": 190, "nozzle_temp_max": 230,
  "tray_type": "PLA",
  "cols": ["RRGGBBAA"], "ctype": 2
}}
```

- For the external spool, `ams_id` is 255 (main) or 254 (deputy), and `tray_id` is
  always 254 (`VIRTUAL_TRAY_DEPUTY_ID`). `cols` and `ctype` are sent only when the
  printer advertises manual multi-colour support (`fun2` bit 23,
  `is_support_filament_manual_multi_color`). `ctype` 2 is a single colour and 0 is
  multi-colour (`DevFilaColorType`). **[High]**
- The printer echoes the command. `MachineObject::parse_json` applies the echoed values
  to the slot and calls `set_hold_count`, so stale `push_status` reports do not roll the
  slot back. **[High]**
- The slot holds a filament ID, colour, temperatures and type. It holds no spool
  identity and no remaining weight. Pressure advance is set separately by
  `AMSTraySettingBase::save_pa_profile_selection`. It goes through
  `CalibUtils::select_PA_calib_result` to `MachineObject::commnad_select_pa_calibration`
  (command `extrusion_cali_sel`), or through `command_extrusion_cali_set` when the
  printer's calibration version is not initialised. **[High]**
- `MachineObject::publish_json` sends through the closed Bambu network plugin
  (`NetworkAgent::send_message` or `send_message_to_printer`). The plugin also exposes
  `install_device_cert` and `check_cert`. Authorization signing is therefore not in the
  open source. A rebuilt Bambu Studio that loads the official plugin is presumably
  "official software" to the printer. **[Medium]**

## 5. Upstream's "manual RFID spool bind" (commit `516b9c8f6`, 2026-09-17)

The Filament Manager (`src/slic3r/GUI/fila_manager/`) keeps spools in
`filament_inventory/spools.json` and syncs them to the Bambu cloud
(`/my/filament/v2/...`). The flow, all **[High]** from code:

1. In `AMSMaterialsSetting`, the user picks a Filament Manager spool for a slot
   (`m_selected_spool_id`).
2. `AMSMaterialsSetting::on_select_ok` sends `ams_filament_setting` with that spool's
   filament ID, setting ID, colour and temperatures, but only for a third-party slot
   (`m_is_third`). It then saves the pressure-advance selection.
3. `wgtFilaManagerStore::force_mount_spool` pins the spool to the device, AMS ID, AMS
   type, slot and (new in this commit) AMS serial number. It sets `mount_hold_count`
   to 5 ticks.
4. `wgtFilaManagerCloudSync::sync_slot_bindings_to_cloud` posts
   `/my/filament/v2/slot-mappings/sync` with `amsSn`, `slotId`, `amsId`, `amsType`,
   `spoolId`, and `rfid` when the spool has one. It skips spools that are not
   cloud-synced, or that have no integer cloud ID or no AMS serial.

What the commit changed:

- `wgtFilaManagerSync::sync_all_trays` no longer reports a "manual RFID bind" through
  the RFID-keyed weight sync (`sync_ams`). A manual RFID bind is a spool record that has
  a valid tag UID, pinned to a slot that reports none, for example a Bambu refill
  on a reused spool, or a tag the AMS did not read. The binding now goes only through
  slot mappings.
- `wgtFilaManagerStore::apply_mount_diff` uses the 5-tick hold so the next
  `push_status` does not eject a fresh binding before the printer reflects it.
- `wgtFilaManagerCloudSync::sync_slot_mappings_to_cloud` no longer sends an empty
  mapping list.

How a binding is kept afterwards (`wgtFilaManagerSync::match_tray`):

- First, the slot pin, while `slot_pin_still_valid` holds. That requires the slot's
  reported filament ID, sub-brand, type and colour to still match the spool.
- Otherwise, the `tray_uuid` RFID.
- Otherwise, for non-RFID spools only, a filament-ID-plus-colour match
  (`find_by_setting_and_color`).
- When the slot stops reporting a spool present, the binding is ejected after the
  hold, and the cloud mapping is unbound.

**Implication.** Upstream already covers a click-to-bind of a spool to a slot. It works
in cloud mode with no extra hardware, and it does not survive unplugging the spool. The
cloud part depends on a Bambu account and on cloud-synced spool IDs.

## 6. Authorization control (Bambu firmware policy)

- P1 series 01.08.02.00 (2025-06-03): "The printer now verifies whether control commands
  originate from official or authorized software. If not, the commands will be
  rejected." Developer Mode, available only in LAN Mode, disables that check.
  Third-party slicers are told to use Bambu Connect. **[High]** (P1 firmware release
  history)
- SpoolEase reads a "locked" bit from the printer's `fun` field (`0x20000000`) and
  publishes no `ams_filament_setting` while the printer is locked
  (`BambuPrinter::is_locked`). **[High]**
- TigerSpool's compatibility table lists the P1S. To write a slot it needs "LAN mode and
  Developer mode"; in cloud mode it is read-only, because "Bambu's cloud does not accept
  the command". **[High]** for the project's own statement.
- OpenSpool's README says only X1C printers on 1.08.05.00 or later need Developer Mode.
  That predates the P1 rollout, and Bambu's P1 release note supersedes it.
  **[Medium]**

## 7. Open projects and standards

| Project | Tag | How it sets the AMS | Reader hardware | Maturity | P1S |
| --- | --- | --- | --- | --- | --- |
| SpoolEase | NTAG215+ (own `SpoolEaseV1` format), reads Bambu tags, OpenPrintTag only when copied to NTAG | Scan at the console, then insert the spool within one minute. Watches `tray_exist_bits` and `tray_reading_bits`, then sends `ams_filament_setting` and `extrusion_cali_sel` (K) | WT32-SC01 Plus (ESP32-S3 with display) and a PN532. Optional ESP32-S3 scale | Active. Release 0.7.0 on 2026-06-22. Apache-2.0 with Commons Clause (free for personal use). Also ships its own customised Bambu Studio build | Listed (P1 line, AMS, AMS 2 Pro, AMS HT). Needs LAN plus Developer Mode to write |
| OpenSpool | NTAG215/216, NDEF `application/json`: `protocol`, `version`, `type`, `color_hex`, `brand`, `min_temp`, `max_temp` | Touch the spool to the reader. The released "Mini" sets the **external spool** (255/254). An experimental AMS build wires two readers to AMS 0, slots 0 and 1. Its AMS PCB is still a TODO | ESP32-S2/S3 (Wemos D1 Mini) and a PN532, ESPHome firmware | Release 1.21.2 on 2025-08-30, repo active in 2026-09. Apache-2.0 software, CERN-OHL-S hardware | Works over LAN MQTT. Needs Developer Mode on current firmware (inferred, see section 6) |
| TigerTag and TigerSpool | NTAG21x binary format, spec v2.2, CC-BY-4.0. Optional ECDSA-P256 origin signature | TigerSpool: tap a slot on its screen, hold the spool to the box, then load the slot it names | ESP32-S3 with a 2-inch touchscreen. Desktop stand or printer clip | Very active (pushes on 2026-10-10). The project claims chips on eSun, Sunlu, Rosa3D and other brands (vendor claim) | P1S listed. LAN plus Developer Mode to write; cloud is read-only |
| OpenTag3D | NTAG215/216, NDEF MIME `application/opentag3d`, fixed memory map, unencrypted, no signature | Standard only. Readers are listed as "in progress" (OpenSpool) or "active implementation" (BambuTagger) | Any ISO 14443A reader (PN532, phone) | Spec v2.003, GPL-3.0. Supporters are small filament brands and tools | Only through a reader project |
| OpenPrintTag (Prusa Research) | NFC-V (ISO 15693, ICODE SLIX2), CBOR in NDEF, no signature by design | Standard only | Needs an ISO 15693 reader. **A PN532 cannot read it**, per SpoolEase's docs | Announced 2025-10-31 (per a forum link). Spec still marked "work in progress" | Only through a reader with NFC-V support |
| BambuTagger | Reads and writes Bambu, TigerTag, SpoolEase, OpenSpool and OpenTag3D | BT-AMS-C is "a four-slot reader that sits on a Bambu Lab AMS" and pushes tag data to the printer | ESP32 builds; the handheld BT-Touch uses an RC522 | Small project (AGPL-3.0). Described from TigerTag's guide only | Not verified |
| Cloned Bambu tags | MIFARE Classic Gen 4 FUID or UFUID with a genuine dump | The AMS reads it natively | A Proxmark3 or Flipper Zero to write, plus magic tags | Documented, community-maintained | Works with no Developer Mode, but only as a Bambu product |

Sources for this table: each project's own repo or docs (see Sources). Confidence is
**[High]** for what each project says about itself. The two claims taken from someone
else's text, the TigerTag adoption figures and the BambuTagger row, are **[Medium]**.

## 8. Recent Bambu firmware and policy changes about third-party tags

- No Bambu announcement or release note adds support for third-party or open tags. A
  forum thread from October 2025 to August 2026 has no staff reply.
  **[High]** for the P1 release notes; **[Low]** for "no announcement", which comes from
  web search.
- 2025-06-03: authorization control reaches the P1 series (section 6). This is the
  policy change that matters most here: it gates every external reader. **[High]**
- By mid-2025: the AMS reportedly writes to block 0 of Gen 2 (CUID) clone tags and
  bricks them (section 1). **[Medium]**
- 2025-08-25 (01.09.00.00): "Optimized RFID reading reliability for the AMS". 2026-03-30
  (01.10.00.00): fixed drying of non-official filaments in the AMS 2 Pro and AMS HT.
  **[High]**
- Bambu Studio, 2026: the Filament Manager auto-registers RFID spools
  (`check_and_register_new_rfid_spools`), adds manual spool-to-slot binding (section 5),
  and stops sending a "set to ?" command when a non-RFID spool is edited, deleted or
  reinserted (commit `ee488c3bc`). **[High]**

## 9. Realistic options for this fork (input to ticket #8)

| Option | Hardware | Printer mode | Trade-off |
| --- | --- | --- | --- |
| A. Manual bind in Studio, extending upstream's Filament Manager bind | None | Cloud or LAN, unchanged | Zero hardware and reuses upstream. Costs one pick per load, and the binding is lost on unplug |
| B. Studio reads a spool tag itself through an off-the-shelf USB NFC reader, then calls `command_ams_filament_settings` | A PC/SC reader and NTAG215 stickers | Unchanged, through the official plugin | Keeps cloud mode. Needs new C++ (PC/SC) in the fork, the reader sits at the PC, and Studio must be running |
| C. Adopt SpoolEase | ESP32 console with PN532, NTAG215 | LAN-only plus Developer Mode | Best AMS workflow (scan then insert, auto slot, K, weight). Loses Handy and cloud printing. Its inventory lives in SpoolEase, and its own Bambu Studio build overlaps with this fork |
| D. Adopt TigerSpool or OpenSpool | ESP32 and PN532 box, NTAG | LAN-only plus Developer Mode | Simple. OpenSpool targets only the external spool; TigerSpool needs a slot tap. Same cloud loss as C |
| E. Cloned Bambu tags | Proxmark3 or Flipper, Gen 4 magic tags | Unchanged | AMS-native, but the spool claims to be a specific Bambu product and colour, so no custom Filament Profile. Clones of one dump share a `tray_uuid`. Tags get bricked if the wrong generation is used. Unsupported by Bambu |

## 10. Questions this raises for the map

- Will the owner run the P1S in LAN-only mode with Developer Mode on, giving up Bambu
  Handy and cloud printing? Options C and D depend on the answer.
- Is one manual pick per spool load acceptable (option A), or must loading be
  zero-touch?
- Would Studio reading tags through an off-the-shelf USB reader (option B) count as
  "adopting an existing open project" or as the out-of-scope "our own reader"?
- If tags win, which format should Spools carry: an NTAG215 format (OpenSpool,
  OpenTag3D, TigerTag, all PN532-readable) or OpenPrintTag (NFC-V, a different reader)?
- The upstream binding's cloud half needs cloud-synced spool IDs. Is that acceptable
  under the plain-files rule? It is still fuzzy whether the fork should use upstream's
  cloud slot mappings or a local-only binding.

## Sources

Upstream code (local clone, `master`):

- `src/slic3r/GUI/DeviceManager.cpp`: `MachineObject::command_ams_filament_settings`,
  `command_ams_user_settings`, `command_ams_refresh_rfid`, `command_ams_refresh_rfid2`,
  `publish_json`, `parse_json`
- `src/slic3r/GUI/DeviceCore/DevFilaSystem.cpp`: `DevFilaSystemParser::ParseAmsTrayInfo`,
  `DevFilaSystem::IsBBL_Filament`
- `src/slic3r/GUI/DeviceCore/DevDefs.h`: `DevAmsType`, `VIRTUAL_TRAY_DEPUTY_ID`
- `src/slic3r/GUI/AMSMaterialsSetting.cpp`: `AMSMaterialsSetting::on_select_ok`,
  `AMSTraySettingBase::save_pa_profile_selection`
- `src/slic3r/GUI/fila_manager/wgtFilaManagerSync.cpp`: `sync_all_trays`, `match_tray`,
  `slot_pin_still_valid`, `check_and_register_new_rfid_spools`
- `src/slic3r/GUI/fila_manager/wgtFilaManagerStore.cpp`: `force_mount_spool`,
  `apply_mount_diff`, `FilamentSpool::is_valid_tag_uid`
- `src/slic3r/GUI/fila_manager/wgtFilaManagerCloudSync.cpp`: `sync_slot_bindings_to_cloud`,
  `sync_slot_mappings_to_cloud`
- `src/slic3r/Utils/NetworkAgent.hpp`: `NetworkAgent`
- `src/slic3r/Utils/CalibUtils.cpp`: `CalibUtils::select_PA_calib_result`
- Commits `516b9c8f6`, `6ba8089c2`, `ee488c3bc`, `ae5df57d3`

Bambu Lab:

- P1 series firmware release history: <https://wiki.bambulab.com/en/p1/manual/p1p-firmware-release-history>
- AMS Lite store page: <https://us.store.bambulab.com/products/ams-lite>
- AMS 2 Pro store page: <https://us.store.bambulab.com/products/ams-2-pro>
- AMS HT store page: <https://us.store.bambulab.com/products/ams-ht>
- AMS HT RFID coil: <https://asia.store.bambulab.com/en/products/ams-ht-rfid-coil>

Tag research:

- RFID-Tag-Guide: <https://github.com/Bambu-Research-Group/RFID-Tag-Guide>
  (`README.md`, `BambuLabRfid.md`, `deriveKeys.py`, `rfid-board/`)
- Co-author fork with newer docs: <https://github.com/queengooborg/Bambu-Lab-RFID-Tag-Guide>
  (`docs/WriteTags.md`)
- Tag dump library: <https://github.com/queengooborg/Bambu-Lab-RFID-Library>
- FM11RF08S backdoor paper: <https://eprint.iacr.org/2024/1275.pdf>

Open projects and standards:

- OpenSpool: <https://github.com/spuder/OpenSpool> (`firmware/bambu.h`
  `generate_mqtt_payload`, `firmware/conf.d/pn532_rfid-solo.yaml`,
  `firmware/conf.d/pn532_rfid-ams.yaml`)
- SpoolEase: <https://github.com/yanshay/SpoolEase> (`core/src/bambu.rs`,
  `core/src/tag_standards.rs`) and <https://www.spoolease.io/docs/quickstart/scan-and-load>,
  <https://www.spoolease.io/docs/quickstart/register-openprinttag-spools>,
  <https://www.spoolease.io/docs/components>
- TigerTag: <https://github.com/TigerTag-Project/TigerTag-RFID-Guide>; TigerSpool:
  <https://github.com/TigerTag-Project/TigerSpool-RFID>
- OpenTag3D: <https://github.com/GooborgStudios/OpenTag3D> (`spec.md`) and
  <https://opentag3d.info/about>
- OpenPrintTag: <https://github.com/OpenPrintTag/openprinttag-specification>
  (`docs_src/nfc_technical_details.md`) and <https://openprinttag.org>
- BambuTagger: <https://github.com/VID-PRO/BambuTagger-Touch>

Forum (low confidence):

- <https://forum.bambulab.com/t/third-party-rfid-status-to-my-surprise/202461>
- <https://forum.bambulab.com/t/rfid-function-in-bambu-studio/202842>
