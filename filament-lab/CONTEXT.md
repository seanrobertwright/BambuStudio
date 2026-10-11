<!-- markdownlint-disable MD013 -->
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
One physical roll of filament, carrying its own colour or colours, batch and remaining amount, and pointing at exactly one Filament Profile.
_Avoid_: roll, reel

**AMS Slot**:
One filament position in an AMS unit, which holds at most one Spool at a time.
_Avoid_: tray, bay

### Calibration

**Calibration Test**:
One printed test whose measured or chosen outcome produces one or more Filament Profile values. Any print-process or machine setting it also suggests, such as the measured cube's wall offsets, is not a Filament Profile value.
_Avoid_: calibration print, tuning test

**Nozzle Flow Type**:
The flow variant of a nozzle, such as Standard or High Flow, which changes some of the values a filament needs at the same nozzle diameter.
_Avoid_: nozzle volume type, hotend type

**Calibration Sequence**:
The ordered list of Calibration Tests run to characterise a new Filament Profile on one printer model, nozzle diameter and Nozzle Flow Type.
_Avoid_: calibration checklist, tuning plan

**Calibration Record**:
The Filament Profile values that a Filament Profile's Calibration Tests produced on one printer model, nozzle diameter and Nozzle Flow Type, applied automatically to every print for that combination. It holds only values that get applied, including pressure advance even though the printer stores it, and never print-process or machine settings; values that do not vary with Nozzle Flow Type, such as shrink, are shared by the records for each flow type.
_Avoid_: calibration profile, calibration preset
