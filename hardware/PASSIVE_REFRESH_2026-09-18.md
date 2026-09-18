# Passive exact-MPN refresh — September 18, 2026

Replaces **51 placed passive instances across 20 legacy Spec identities** with
verified exact-MPN parts: 38 on Power and 13 on Microcontroller. Existing exact-MPN
R10 (`SEWF3920D5L00P9`, P-00036) is unchanged. The replacements follow the recorded
OCPM migration mapping, not a value-based substitute search.

Library pin: `53fd089c5db5ecc82db927eae2bab66f04c0f0e9` →
`8a4c96de2d5a9d68c17b1cda6ae49fe697b1eeee`. The new commit was confirmed present on
the public library's remote main. Application and managed OCPM repositories were
not modified by this design migration.

## Replacement list

| References | Old IPN | New IPN | Exact orderable MPN |
|---|---|---|---|
| C1, C19, C25, C27, C28, C29, C32, C34, C38 | P-00012 | P-00061 | GRM188R71E104KA01D |
| C2, C23, C26 | P-00013 | P-00054 | GRM21BR71A225KA01L |
| C3 | P-00014 | P-00060 | GRM188R71C224KA01D |
| C4, C6, C8, C10, C12, C14 | P-00015 | P-00062 | GRM32ER72A225KA35L |
| C16 | P-00016 | P-00055 | GCM21BR72A104KA37L |
| C5, C7, C9, C11, C13, C15, C20, C21 | P-00017 | P-00063 | GRM32ER71E226KE15L |
| R3, R4 | P-00018 | P-00056 | RC0402FR-071RL |
| R1, R2, R7 | P-00019 | P-00057 | RC0402FR-0710KL |
| C17 | P-00020 | P-00068 | GRM21AR72A224KAC5L |
| C18, C30, C33, C37 | P-00021 | P-00047 | GRM21BR71E105KA99L |
| R5 | P-00022 | P-00058 | RC0402FR-07100KL |
| R6 | P-00023 | P-00059 | RC0402FR-0724K9L |
| C22, C24 | P-00024 | P-00069 | 12101C475KAT2A |
| C31 | P-00028 | P-00026 | C2012X7R1A106K125AC |
| C35, C36 | P-00029 | P-00048 | 04025A6R8JAT2A |
| R8 | P-00033 | P-00049 | TNPW0805402KBEEA |
| R9 | P-00034 | P-00050 | TNPW0805100KBEEA |
| R12 | P-00043 | P-00053 | CRCW20102K20FKEFHP |
| R14 | P-00044 | P-00065 | RT0402BRD0744K2L |
| R11, R13 | P-00045 | P-00066 | RC0402JR-070RL |

## Verification performed

- Exact reference/UUID set agrees with the migration ledger for all 51 instances.
  All selected target records are verified, active exact-MPN Parts.
- Nominal values match numerically, including unit spellings; capacitor dielectric
  and minimum voltage, resistor tolerance/TCR/power/voltage bounds checked against
  the old fields. Zero-ohm jumpers use maximum resistance/rated current instead
  of inapplicable tolerance/TCR. Exact rating conditions copied from the library.
- All replaced symbol pin numbers, electrical types, positions, orientations and
  lengths match the old symbols. Every instance UUID, position/orientation,
  mirror/unit setting, pin UUID, DNP/BOM/board flag and reference is unchanged.
- Every replaced cached symbol matches the new pinned library. Instance metadata
  matches the exact-MPN library fields while retaining existing field positions;
  new manufacturer/technical fields are hidden. Visible values may use the
  library's equivalent notation (for example `2.2k` instead of `2.2 kΩ`).
- Shared footprint identities and engineering hashes are unchanged; associated
  STEP model files are byte-identical between pins. R12's shared 2010 footprint
  only has its previously approved ownership/version metadata revision.
- Independent KiCad 10.0.6 XML netlist exports have **identical net names and
  reference/pin membership**: Power 52 components / 34 nets, MCU 16 / 43.
- Full ERC finding objects are identical before/after, not merely equal counts:
  Power **17 errors / 4 warnings**; MCU **38 errors / 30 warnings**. These are
  pre-existing per-root diagnostics, not a clean-design or multi-root signoff.
- All wires, junctions, labels, graphics and non-replaced instances are unchanged.
  Overview schematic, project settings, library tables and PCB are byte-identical.
  PCB is an empty layout scaffold: there is no routing or populated placement to
  update. C35/C36 remain DNP as before.
- After-sheet SVG previews were rendered and visually inspected. No new labels,
  wires or unrelated circuitry were moved to accommodate the replacement.

## Engineering boundaries retained

- C17 uses the selected Murata GRM21AR72A224KAC5L (general-purpose), not the earlier
  unconfirmed automotive GCM candidate. Nominal 220 nF / 100 V / X7R / 10% / 0805
  requirements match; this does not claim automotive qualification.
- C22/C24 use the selected KYOCERA AVX 12101C475KAT2A, not the unconfirmed Murata
  candidate. Maximum body height is 2.79 mm; the unchanged generic STEP is not a
  guaranteed maximum-clearance envelope.
- Voltage/current/power ceilings and temperature derating remain simultaneous
  limits, not independent operating promises. Capacitor DC-bias behavior and
  project thermal/pulse/mechanical qualification are not established by swapping
  verified library identities. Existing design issues remain separate work.

This is a reviewable design update, not a fabrication release or new approval of
library parts. No component records or verification receipts were changed.
