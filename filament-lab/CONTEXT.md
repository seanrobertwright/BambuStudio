# Filament Lab

Fork-only work on filament profiles, filament calibration and custom-spool identification for Bambu Studio printers with an AMS.

## Language

### Filament

**Vendor**:
The brand that sells a filament product line, whether or not Bambu Studio ships profiles for it.
_Avoid_: provider, manufacturer

**Filament Profile**:
The settings for one Vendor's filament product line (for example "SUNLU PLA+"), independent of colour and batch.
_Avoid_: filament type, material profile

**Spool**:
One physical roll of filament, carrying its own colour, batch and remaining weight, and pointing at exactly one Filament Profile.
_Avoid_: roll, reel, tray

**AMS Slot**:
One filament position in an AMS unit, which holds at most one Spool at a time.
_Avoid_: tray, bay

### Calibration

**Calibration Test**:
One printed test whose measured or chosen outcome produces one or more Filament Profile values.
_Avoid_: calibration print, tuning test

**Calibration Sequence**:
The ordered list of Calibration Tests run to characterise a new Filament Profile on one printer model and nozzle size.
_Avoid_: calibration checklist, tuning plan

**Calibration Record**:
The values a Filament Profile's Calibration Tests produced on one printer model and nozzle size, applied automatically when slicing for that pair.
_Avoid_: calibration profile, calibration preset
