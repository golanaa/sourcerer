# Sourcerer hardware

Open `pcb/sourcerer_v1.kicad_pro` with KiCad 10.0.4 or later.
`pcb/` is the actual working design tracked by Git, not an exported copy.
The STM32CubeMX configuration is `../firmware/sourcerer_v1_1.ioc`.

## Component dependencies

`libs/` contains only the symbol, footprint and 3D-model dependencies selected
for this project. The project library tables and its `OCPM_3DMODEL_DIR` text
variable resolve them locally; no external workspace or global OCPM setup is
required. Open the project, rather than a detached board, to load these settings.

These are versioned dependencies. Add or revise a part through the component
workflow and deliberately update the project's dependency set as part of that
design change. Routine commits do not regenerate, export or scrub the design.
Project-local tables intentionally resolve the pinned parts, not whatever newer
version happens to be installed globally. Do not edit the pinned libraries as
an independent component source.

The selected symbol blocks and footprint/model files retain their source content,
engineering hashes and metadata. `libs/dependencies.json` records the bundled
file hashes and selected symbol hashes. Embedded schematic symbols preserve the
working design; updating them from a library remains an explicit engineering edit.
The dependency set is not a complete component-management database or approval
history. Source-manifest fields refer to the originating component library.

## Development state

This is unfinished hardware, not a fabrication release. The PCB is currently an
empty layout scaffold; the CubeMX configuration is peripheral planning, not
completed closed-loop firmware.

At the 2026-09-09 directory cleanup, the working design exported 52 components
and 34 nets, unchanged by the cleanup. ERC reported 21 existing findings:
9 unresolved text variables, 6 undriven power pins, 2 unconnected pins,
3 isolated-label warnings and 1 multiple-net-name warning. Local power-symbol
dependencies resolve the previous host-library warnings; no circuit fixes or
component approvals are implied by this directory change.

## Third-party material

The libraries include KiCad material and project-specific adaptations. Preserve
KiCad's CC-BY-SA 4.0 terms and design exception in
`libs/KICAD-LIBRARIES-LICENSE.md`, and the attribution headers in model files.
Original KiCad collections:

- https://gitlab.com/kicad/libraries/kicad-symbols
- https://gitlab.com/kicad/libraries/kicad-footprints
- https://gitlab.com/kicad/libraries/kicad-packages3D

EPC2305, LMG1210 and RB568 models are project-generated visual approximations.
The SEWF3920 model is manufacturer-supplied C&B / RESI CAD from
https://www.cb-resi.com/seriesInfo?id=265 . Other model files retain their original
source notices. This notice does not assign a new license to the Sourcerer design
or relicense manufacturer-supplied material.
