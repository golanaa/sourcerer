# Sourcerer hardware

Open `pcb/sourcerer.kicad_pro` with KiCad 10.0.4 or later, with the standard
KiCad symbol and footprint libraries installed. This is the working design,
not an export. The CubeMX configuration is `../firmware/sourcerer.ioc`.

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

### Pinned library revision

[`ocpm-library.lock`](ocpm-library.lock) records the library repository and exact
commit expected by this Sourcerer revision. Use that commit, not whichever
library version happens to be newest. The lock file documents the dependency;
KiCad does not read or enforce it automatically.

After cloning Sourcerer, run these commands **from the Sourcerer repository root**
to create a separate sibling library checkout and select the pinned revision:

```sh
git clone https://github.com/golanaa/ocpm-library.git ../ocpm-library
git -C ../ocpm-library checkout --detach "$(sed -n 's/^commit=//p' hardware/ocpm-library.lock)"
```

If the sibling checkout already exists, skip cloning. After switching Sourcerer
branches or pulling changes, check its lock file and select the recorded revision:

```sh
git -C ../ocpm-library fetch origin
git -C ../ocpm-library checkout --detach "$(sed -n 's/^commit=//p' hardware/ocpm-library.lock)"
```

These commands use a detached checkout to reproduce the recorded library
revision. Create a library branch before making component changes; preserve any
local work before switching revisions.

### Updating the dependency

When a design change needs new or revised library components, commit and publish
those components in the library repository first. Update the full `commit=` hash
in `ocpm-library.lock` and commit that pin alongside the Sourcerer design change.
The pinned commit must be available from the recorded repository. Ordinary design
changes that use the same library revision do not require a pin update.

Sourcerer contains no copied component libraries or Git submodule. The separate
checkout remains the library source; updating the pin does not copy any files.

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
