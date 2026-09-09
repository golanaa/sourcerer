# Sourcerer hardware

Open `pcb/sourcerer.kicad_pro` with KiCad 10.0.4 or later, with the standard
KiCad symbol and footprint libraries installed. This is the working design,
not an export. The CubeMX configuration is `../firmware/sourcerer_v1_1.ioc`.

## Schematic sheets

The project uses three flat top-level sheets:

1. **Sourcerer** (`sourcerer.kicad_sch`) - editable overview and general information.
2. **Power** (`power.kicad_sch`) - power conversion and supporting circuitry.
3. **Microcontroller** (`microcontroller.kicad_sch`) - controller and supporting circuitry.

The overview contains only graphics and text; it adds no electrical connections.
Open the project to load the complete sheet set.

## Separate shared-library checkout

Keep the Sourcerer checkout and shared component-library checkout beside one
another. Use `ocpm-library` as the directory name for the library checkout:

```text
work/
├── sourcerer/
│   └── hardware/pcb/sourcerer.kicad_pro
└── ocpm-library/
    ├── verified/
    │   ├── verified.kicad_sym
    │   ├── verified.pretty/
    │   └── verified.3dshapes/
    └── unverified/
        ├── unverified.kicad_sym
        ├── unverified.pretty/
        └── unverified.3dshapes/
```

The project library tables and `OCPM_3DMODEL_DIR` project text variable point
to that sibling directory. No personal absolute path or manual global-library
registration is required. Open the project rather than a detached board so
KiCad loads the project settings. Reopen an already-open project after changing
these paths.

Sourcerer contains no copied component libraries, dependency-export manifest,
Git submodule, or per-commit library synchronization step. Pull the shared-library
repository when you want its latest components. On the OCPM host, this same
sibling path may be a local link to the authoritative library instead of another
checkout. The separate repository and its publication workflow are being set up
independently; this directory change does not create or publish that repository.

Placed symbols and footprints remain embedded in the schematic and board.
Applying library revisions to them is still an explicit KiCad action. Existing
parts do not change merely because you pulled a newer library. `Unverified`
components remain candidates; availability is not an approval.

## Development state

This is unfinished hardware, not a fabrication release. The PCB is currently an
empty layout scaffold, so assigned footprints must still be obtained from the
shared library when populating it. The CubeMX file is peripheral planning, not
completed closed-loop firmware. Existing circuit/ERC findings remain design
work; no circuitry or approval states are changed by this directory setup.
