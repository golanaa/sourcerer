# Sourcerer v1 System Architecture

**Publication scope:** This repository currently publishes the requirements and
architecture documents only. References to other working notes, plans, and
editable diagrams name unpublished project artifacts.

**Schematic/IOC update — 2026-09-10:** `LOAD_V_SENSE` is assigned to
PB13 / `ADC3_IN5` in both the schematic and IOC. A fresh MCU netlist confirms
U4 pin 27 (PB13) as the only node on that net. The delivered-load
divider/filter/protection circuit and acquisition-time validation remain pending.
All IOC conversion and timing settings are unchanged. The external INA241
reference buffer is selected but not yet present in the synced power schematic.

_Status: reviewed draft, aligned with the approved v1 rated electrical envelope_

**Documentation review:** 2026-09-10 — current filenames, implementation notes, and publication state reconciled; no new electrical limits or protection policy selected.

## 1. Purpose

This document maps the Sourcerer v1 requirements into an initial system architecture.

It is intended to sit between:
- `sourcerer_requirements.md`
- the current schematic, PCB, firmware, interface, bring-up, and verification artifacts

This document should answer:
- what the major system blocks are
- which block owns which behavior
- how the converter behaves at a state-machine level
- how sensing, configuration, fault handling, interfaces, and host-mediated miner coordination fit together
- what remains intentionally open for later detailed design

It should not yet lock:
- exact component choices
- exact switching frequency
- exact compensation/control-loop implementation
- exact command packet format
- exact backend-neutral supervisory-UART packet framing
- exact AxeOS or Mujina host-backend implementation
- exact thermal derating curves or trip thresholds

---

## 2. Architecture Goals

The v1 architecture shall support a power-supply-first solar MPPT reference design that:
- behaves as a safe solar-input buck power supply before acting as an optimizer
- supports constrained source optimization without disruptive wide-range sweeps during normal load operation
- powers a Bitaxe-focused downstream load
- demonstrates basic Bitaxe coordinated control through a backend-neutral supervisory UART and external host
- preserves first-PCB hardware for solar-first wall assist without making it a prerequisite for standalone converter bring-up
- remains bench-usable, reproducible, and understandable by another engineer

The design should be structured so that:
- real-time safety stays local to the converter controller
- host or miner communication failure does not compromise supply safety
- future architecture work can expand coordination without forcing a v1 redesign

---

## 3. Top-Level System Blocks

The v1 system is organized around these blocks:

- PV / DC input source
- input protection and EMI/filtering network
- PV-source current sensing after input protection and before the input-capacitor and auxiliary-power branches
- passively ORed PV/wall auxiliary-power input
- auxiliary 5 V regulator and 3.3 V post-regulator
- auxiliary-rail power-good/startup interlock
- non-isolated buck power stage
- gate driver and switch control path
- output filter and programmable output rail operating within the rated electrical envelope
- reverse-current blocking on the MPPT source branch
- optional external isolated wall-supply DC input, physical disconnect, controller-commanded branch isolation, branch protection, and reverse-current blocking
- pre-load combined source bus
- controller-owned common load disconnect / protection stage
- separate PV-source, MPPT-inductor, wall-branch, and total-load current observables
- voltage and temperature sensing
- STM32G4-class converter controller
- nonvolatile configuration/calibration storage
- backend-neutral supervisory-host UART interface
- debug/development UART interface
- local status/control elements
- Raspberry Pi-class external coordination host with modular miner-control backends
- connected Bitaxe board for the v1 coordinated-control demo

Companion diagram:
- Editable source: `../diagrams/sourcerer-system-overview_simplified.drawio` (not published)
- [Public README image](../assets/sourcerer-system-overview.png)

This is a simplified functional view, not a netlist or a completeness checklist.
Auxiliary power-output wiring is intentionally omitted; PGOOD remains shown as
a status signal. The combined host/miner-interface box represents host-side API
adaptation, not AxeOS firmware running on the Pi. Detailed signal meanings and
required measurements are defined below.

The STM32G4 controller is the center of the v1 power architecture. It owns converter control, protection, source and load telemetry generation, local configuration behavior, and the backend-neutral supervisory interface. The external host owns miner-specific API adaptation and experimental source-aware coordination policy.

---

## 4. Control Partition

### 4.1 STM32G4 Converter Controller

The STM32G4 shall own all fast and safety-relevant converter behavior:
- PWM generation
- ADC sampling coordination
- local output regulation
- constrained MPPT/source-tracking behavior
- input/output current and voltage limiting
- fault detection
- output enable/disable control
- soft-start behavior
- source-limited behavior
- overtemperature handling
- nonvolatile configuration validation
- local status indication

The v1 controller clock architecture shall use the fitted 24 MHz HSE crystal as its primary reference. The final PLL, HRTIM, ADC, and peripheral clock configuration shall be established during the intentional CubeMX/peripheral configuration pass. The active IOC's present HSI-based scaffold is not the final clock architecture.

The controller shall remain capable of safe converter operation if:
- the supervisory host is absent
- host commands are stale, invalid, or unsupported
- AxeOS, Mujina, or miner communication is unavailable
- the debug UART is disconnected
- host-side logging software is not running

### 4.2 Backend-Neutral Supervisory Interface

For v1, the MPPT controller communicates over a wired UART with a Raspberry Pi-class external host. The converter firmware does not act as an AxeOS, Mujina, or other miner-specific client.

The supervisory interface shall support:
- reporting Sourcerer source, rail, load, mode, limit, and fault telemetry
- accepting bounded configuration and operating commands within the rated electrical envelope
- reporting supervisory-host connection and command validity/status
- rejecting invalid commands while retaining the previous valid setting
- detecting stale host control and entering defined conservative behavior without compromising local regulation or protection

The exact packet framing, freshness mechanism, and host command lease behavior remain to be defined in later interface and software design work.

### 4.3 External Coordination Host and Miner Backends

The v1 external host shall provide a modular boundary between the Sourcerer supervisory protocol and miner-specific control APIs.

The initial demonstration path is:

1. Sourcerer supervisory UART to a Raspberry Pi-class host
2. host source-aware coordination logic
3. host AxeOS backend over the network
4. one Bitaxe running stock AxeOS

Mujina is an alternate and strategic future backend. A Mujina-managed Bitaxe may instead run `bitaxe-raw` and connect to the host using the transport required by Mujina. That miner-side transport is outside the Sourcerer converter-firmware architecture.

The host may read miner telemetry and command miner operating parameters, but it shall not be part of the converter's fast regulation or protection loops. Loss of the host, network, miner backend, or miner telemetry shall not prevent safe converter operation.

The architecture of a possible future UART-to-Wi-Fi/Ethernet gateway is intentionally open. Experience with the Pi implementation shall determine whether such a gateway owns policy or allocation, or only translates and routes the neutral Sourcerer protocol to AxeOS and/or Mujina APIs.

### 4.4 Debug / Development Host

The debug/development interface supports:
- firmware programming and debug
- telemetry/status logging
- diagnostics
- development-time command/control

The debug/development host is not required for normal safe converter operation.

---

## 5. Operating Mode Model

The firmware architecture should expose a small set of clear operating modes.

Initial v1 mode model:

- **OFF / DISABLED**
  - output disabled by configuration, local control, external command, or fault handling

- **STARTUP**
  - validate configuration
  - establish initial sensing sanity checks
  - perform soft-start if enabled
  - transition into the configured operating mode when safe

- **REGULATE_ONLY**
  - maintain output regulation without active MPPT exploration
  - factory-default startup mode
  - preferred fallback mode when source/load coordination is unavailable

- **SAFE_TRACK**
  - perform bounded source optimization while preserving output-rail constraints
  - perturbations are local and reversible
  - exploration is inhibited when rail margin or source condition is poor

- **SOURCE_LIMITED**
  - entered when the source cannot support the commanded operating point within valid limits
  - output behavior is controlled rather than unpredictable
  - telemetry indicates source-limited status

- **FAULT**
  - output disabled while active fault is present
  - recovery behavior depends on automatic-retry or latched fault class

The exact transition conditions, debounce times, retry timers, and state-machine encoding remain architecture/design work.

Wall-assist presence is an orthogonal source condition, not a replacement operating mode. The controller may therefore be in `REGULATE_ONLY`, `SAFE_TRACK`, `SOURCE_LIMITED`, or `FAULT` while independently reporting PV availability, wall availability, and the contribution of each source.

---

## 6. Sensing and Calibration Architecture

### 6.1 Required Measurements and Their Meanings

The v1 sensing architecture shall support measurement or validated derivation of:
- PV-source input voltage
- total current drawn from the PV source
- MPPT-branch inductor current
- local MPPT converter-output voltage after the output filter and before the MPPT reverse-current block
- output/load voltage after the common load disconnect
- total current delivered to the Bitaxe/load after the common load disconnect
- pre-disconnect combined-bus voltage
- wall-input voltage and wall-branch current when wall assist is populated
- at least one thermally meaningful converter temperature

These current quantities shall remain distinct:
- **PV-source current** is the total current drawn from the PV source, including the main converter and auxiliary-power branch.
- **MPPT-inductor current** is the continuous current produced by the buck power stage before output capacitance and source combining.
- **Wall-branch current** is the wall source's contribution to the combined bus.
- **Total load/output current** is the current delivered to the Bitaxe/load after the common load disconnect.

The implementation used to obtain total load/output current may be a direct measurement or a validated derivation from branch measurements. That choice remains deferred until the common load-disconnect and protection architecture is selected.

The local MPPT converter-output voltage is required for the fast control and protection loop. It is not mandatory external telemetry, although it may be exposed for development and diagnostics.

The output/load voltage after the common load disconnect represents the voltage actually delivered to the Bitaxe/load and is the externally reported output voltage.

Derived telemetry shall include:
- PV input power
- MPPT-branch power
- output/load power
- wall-input power and source-contribution status when wall assist is populated
- common load-disconnect state and fault status
- current operating state and mode
- source-limited status
- fault class and fault type

### 6.2 Current-Sense Placement

The PV-source current shunt shall be placed after input protection but before the input capacitors and auxiliary-power branch. It shall use an external high-side current-sense amplifier rated for the complete documented input common-mode and transient environment.

The MPPT-inductor current shunt shall be placed after the inductor and before the MPPT output-capacitor bank and branch-combining point. This quiet high-side location measures continuous inductor/MPPT-branch current without disturbing the system ground.

The v1 MPPT-inductor current path shall use an external high-side current-sense amplifier.

A possible future implementation using two STM32 internal op-amps—an attenuating difference stage followed by a gain stage—may be reconsidered only if the final STM32G474 pinout and analog routing remain suitable after the design is more complete. It is not part of the v1 baseline and shall not compromise required sensing, protection, interface, or debug resources.

Wall-branch current shall be measured within the wall branch at a location that reports wall-source contribution without including MPPT-branch current.

The total-load-current implementation remains deferred until the common load-disconnect/protection device and topology are selected. The architecture shall not assume that an unselected load-disconnect device provides a current-monitor output.

### 6.3 Voltage-Sense Placement

The local MPPT converter-output voltage shall be sensed after the output filter and before the MPPT reverse-current block. This is the fast voltage-loop feedback and protection measurement.

Combined-bus voltage shall be sensed before the common load disconnect for source-combining supervision and wall-voltage compatibility checks.

Output/load voltage shall be sensed after the common load disconnect. This is a supervisory measurement used for delivered-voltage telemetry and output undervoltage behavior. Its schematic/IOC assignment is PB13 / ADC3_IN5 (`LOAD_V_SENSE`), replacing optional
TEMP_SPARE. The divider/filter/protection network remains to be implemented
and validated for the 2.5 V ADC reference and actual fault/transient envelope;
PA3 converter-temperature sensing is retained.

### 6.4 Sampling and Fast-Protection Routing

Local MPPT converter-output voltage shall remain on `PA2 / ADC1_IN3`.

The conditioned MPPT-inductor current signal shall use `PA7 / ADC2_IN4`. This allows local MPPT converter-output voltage and MPPT-inductor current to be sampled simultaneously from separate ADC instances.

The precision shunt/amplifier path is not assumed fast enough for severe
protection. The working Rev-A fast-sense direction is a protected low-side
RDS(on) node shared by COMP1 (predictive OCP) and COMP4 (ZCS), with independent
externally divided DAC thresholds. Pin assignments are recorded in the pin
budget; circuit values and implementation status belong in the hardware ledger.

Low-side sensing cannot observe the high-side current rise. The predictive
protection invariant shall bound valid pre-pulse current plus the maximum
possible rise during one authorized high-side pulse, including a sudden output
short, using minimum credible inductance and maximum applied voltage. If that
bound cannot be established, an alternative qualified protection mechanism is
required. PA12 / HRTIM1_FLT1 remains available for a protected external
comparator/fault source. The final threshold, event handling, and state-machine
implementation are not defined by this documentation update.

No high-side pulse permission may be inferred from an invalid/recovering
low-side signal, a missing valid decision, or an unready reference. Positive
clamp recovery can falsely indicate low current. ZCS qualification must also
reject that recovery and switching transients.

**Sensing decisions — 2026-09-10:**

PV voltage PA0 and PV current PA1 may be acquired sequentially on ADC1;
simultaneous PV sampling is **not required**, per Tyler's 2026-09-10 decision.
Filtering, sample spacing, and averaging must still support calibrated PV
power/source telemetry. Preserve the separate-ADC PA2/PA7 fast-control pair.

PB13 / ADC3_IN5 is assigned in schematic and IOC to delivered-load voltage (`LOAD_V_SENSE`),
replacing optional TEMP_SPARE. PA3 remains TEMP_CONVERTER. This removes the
pin-selection blocker for load voltage, not the remaining circuit and sampling
work. The fresh MCU netlist confirms LOAD_V_SENSE at U4 pin 27 (PB13),
with no sensing circuitry yet connected to that net.
Any direct total-load-current channel remains a separate implementation choice.

The INA241 reference will use an **external buffer**, not internal OPAMP2.
PA6 ZCS_DAC and PB14 wall-current sensing remain allocated. Select and validate
the buffer circuit, nominal 0.5 V reference, offset/headroom, stability, and
startup behavior before treating the precision channel as complete. The
external buffer is not yet present in the 2026-09-10 synced power schematic.

Wall voltage, wall current, and combined-bus voltage remain supervisory ADC4
measurements on PB12/PB14/PB15. The updated IOC pin allocation and ADC sequence
counts survive CubeMX load/save, but synchronized triggers, DMA, HRTIM timing,
clock/reference initialization, and complete fault behavior remain scaffold
work. The repeated VP_RIF virtual-pin diagnostic is not a clean-warning signoff.

### 6.5 Calibration Model

Calibration data shall be stored nonvolatilely and validated before use.

The architecture shall distinguish:
- operational configuration
- calibration data
- factory defaults

Calibration-only changes shall not assert the factory-default-versus-modified operational configuration indicator.

---

## 7. Power-Stage Architecture

The v1 power stage is a non-isolated synchronous buck converter operating within the authoritative rated electrical envelope defined by `sourcerer_requirements.md`.

The architecture shall not duplicate the numeric voltage, current, and power limits from that envelope.

The first-prototype switching-frequency baseline is 400 kHz. Operation at 500 kHz remains optional and shall not be enabled until semiconductor, magnetic, waveform, loss, and thermal validation supports it.

The architecture shall explicitly handle buck dropout and source-limited behavior. It shall not imply boost capability.

A longer minimum off-time or reduced switching frequency near dropout was
explored on 2026-09-08 to allow sense-node recovery; neither is selected.
The rated envelope is unchanged. Any such implementation requires renewed
ripple, maximum-on-time/short-circuit, loss, and control validation.

Detailed design work shall still define or validate:
- the remaining PV-source/wall sensing components and the selected MPPT-inductor
  channel's reference, filtering, headroom, and validation
- compensation and control-loop implementation
- final protection thresholds and timing
- snubber, damping, and filtering provisions
- final component thermal and transient margins

### 7.1 Auxiliary-Power Architecture

The auxiliary-power path shall operate independently of main-buck startup.

Raw PV and raw wall inputs shall be passively ORed into `+AUX_IN` upstream of the controller-owned wall-branch isolation stage. This allows the controller to start from PV only, wall only, or both sources without creating a firmware bootstrap dependency.

`+AUX_IN` shall feed the LMR36520 5 V regulator. The 5 V rail shall power the GaN gate driver and feed the LP5907 3.3 V post-regulator used by the MCU and analog circuitry.

The LMR36520 enable input shall use an external divider implementing the nominal startup and shutdown behavior required by `REQ-SAFE-012`. The existing 5 V feedback divider is separate from this new enable divider.

The LMR36520 open-drain `PGOOD` output shall be pulled up to the downstream 3.3 V logic rail and connected to the MCU as the auxiliary 5 V rail-valid signal. `PA15` is assigned to this signal as `PG_5V0` in the updated schematic labels and IOC; `PA6` is the ZCS DAC output.

Firmware shall not arm HRTIM PWM outputs until:
- the MCU has completed reset and startup validation
- `PGOOD` is asserted
- required sensing and configuration checks have passed
- no active fault prohibits operation

`PGOOD` is a startup interlock and diagnostic signal, not the sole auxiliary-collapse shutdown mechanism. Loss of auxiliary power shall also result in safe behavior through:
- LMG1210 VDD undervoltage lockout
- MCU brownout/reset behavior
- inactive HRTIM output states
- gate-input pull-down resistors

The saved schematic and IOC use `PG_5V0` on MCU `PA15`, with R7 = 10 kΩ
pulling up to 3.3 V. `PA6` is the ZCS DAC output, not gate-driver enable or
PGOOD. The separate EN divider is R8 = 402 kΩ / R9 = 100 kΩ, using verified
OCPM Specs `P-00033`/`P-00034`. Runtime interlock behavior, guaranteed
threshold bounds, and auxiliary startup/collapse validation remain open.
Current implementation and check scope are tracked in `../PROJECT_STATUS.md`.

### 7.2 Wall-assist source-combining architecture

The first custom PCB shall implement this power-path structure:

1. PV input → input protection → buck/output filter → MPPT-branch reverse-current block
2. external isolated wall supply DC → removable disconnect and controller-isolatable branch protection → wall-branch reverse-current block
3. both branches join at a pre-load combined bus
4. the combined bus feeds the Bitaxe through a controller-owned common load disconnect/protection stage

The AC/DC supply and all mains wiring remain external to the PCB. The wall connector accepts only its isolated low-voltage DC output.

The common load disconnect is required because disabling the buck alone cannot remove miner power while the wall source is present. It shall provide a hardware path for local disable and fault shutdown while leaving the controller powered from the pre-disconnect bus when practical. Reverse blocking on both branches is separate from this load-disconnect function and prevents a healthy source from energizing the other source connector or power stage.

The wall branch also has a controller-owned electrical enable/isolation path. This is distinct from its physical service disconnect and automatic reverse blocking. It permits controlled source sequencing, wall-branch fault isolation, and safe use of low output setpoints that are incompatible with the fixed wall-supply voltage. On any incompatible wall-voltage/output-setpoint combination, the common load disconnect stays open until the wall branch is isolated and the bus is within the allowed rail envelope.

Solar-first control is priority sharing rather than equal current sharing:
- the wall supply operates as a lower-voltage CV floor
- the MPPT branch injects the available solar-derived current/power into the combined bus
- while the wall supply is holding the bus, the PV controller regulates source/inductor current or power under MPPT command rather than fighting the wall supply as a second stiff voltage regulator
- as PV contribution rises, wall current naturally falls
- when PV can support the full load, the wall branch reaches zero current and its reverse block turns off
- if PV power exceeds load demand, the MPPT branch transitions to the bus-voltage ceiling and curtails PV rather than overcharging the bus

The exact wall setpoint is not fixed yet. Its worst-case upper value, the MPPT bus ceiling, reverse-block drops, load-disconnect drop, regulation tolerances, and explicit handoff hysteresis must leave a documented non-chatter margin.

Startup and source-loss behavior:
- **PV only:** normal MPPT soft-start and standalone operation
- **wall only:** wall source establishes the combined bus; buck PWM remains disabled until PV validity checks pass; the common load disconnect may then soft-connect the miner
- **wall first, then PV:** ramp MPPT current contribution from zero so wall current falls monotonically without two voltage loops fighting
- **PV first, then wall:** verify wall voltage and reverse-block state before accepting wall availability; the wall branch remains off until bus droop requires it
- **loss of PV:** wall supply takes over through the designed voltage offset and transient margin
- **loss of wall:** PV continues if it can support the load; otherwise the existing source-limited/undervoltage behavior applies
- **neither source:** load disconnected and controller unpowered or in its defined brownout state

The wall-only and both-source sequences apply only to a load-rail setpoint compatible with the detected fixed wall voltage. Lower-voltage operation, including 5 V-class Bitaxe testing, uses PV-only operation with the wall branch electrically isolated or physically disconnected.

Initial bring-up shall occur with the wall branch physically disconnected. Wall-assist validation is a later bring-up phase and shall not gate basic buck-control learning or standalone regulation.

---

## 8. Bitaxe Integration and v1 Demo Model

The v1 Bitaxe integration is intentionally narrow.

The architecture shall support:
- one connected Bitaxe/load in v1
- safe supply operation without upstream Bitaxe firmware changes
- reporting Sourcerer telemetry to an external host over the backend-neutral supervisory UART
- reading selected Bitaxe telemetry through a modular host-side miner backend
- commanding at least one power-relevant Bitaxe operating parameter through that backend
- demonstrating an observable load-relevant effect from the commanded Bitaxe parameter

Acceptable v1 control candidates include:
- mining enable/disable as a minimum viable control target
- frequency setpoint if available and safe

The initial backend uses the network API of stock AxeOS. The exact AxeOS telemetry fields, command mapping, and required operating parameter remain open until measured behavior is verified. The host architecture shall permit a Mujina backend without changing the Sourcerer UART protocol.

If the host is absent, its command is stale or invalid, or miner behavior is unknown, the converter shall remain within a conservative locally safe operating state and shall not rely on miner-side cooperation for protection.

---

## 9. External and Debug Interface Model

### 9.1 Supervisory-Host UART

The supervisory UART is a backend-neutral connection between the MPPT controller and an external host or future gateway.

Its v1 architectural role is:
- Sourcerer telemetry and status reporting
- bounded Sourcerer configuration and operating commands
- host connection, command validity, and freshness reporting
- transport of the information required for host-mediated source-aware coordination

AxeOS, Mujina, miner discovery, multi-miner allocation, and network communication remain host-side concerns and are not encoded as converter-firmware dependencies.

### 9.2 Debug / Development UART

The debug UART is a separate development surface.

Its v1 architectural role is:
- logging
- diagnostics
- development-time command/status access
- possible interactive console access if later selected

Interactive console support remains open.

### 9.3 Minimum Command/Status Model

The exact packet/protocol definition remains open, but the architecture shall provide a path for:
- telemetry/status reporting at a useful real-time rate, with the current working target defined by `REQ-INTF-016`
- output voltage setpoint
- output undervoltage shutdown margin
- output current limit
- output power limit
- operating mode selection
- output enable/disable
- fault clear/reset
- wall-source voltage/current/power and source-presence status
- common load-disconnect command/status

Runtime commands requesting output voltage, output current limit, or output power limit outside the rated electrical envelope shall be rejected atomically. The previous valid setting shall be retained, and the rejection reason shall be reported.

This differs intentionally from startup handling of invalid persisted settings, which are clamped to safe allowed bounds before use.

---

## 10. Fault and Protection Architecture

The fault architecture shall support two fault classes:

- **Automatic-retry fault class**
  - clears automatically when the causing condition is no longer present
  - examples include insufficient source, input undervoltage, source-collapse-related output undervoltage, overtemperature after cooldown, and input overvoltage after return to range

- **Latched fault class**
  - remains asserted until its defined recovery action
  - examples include reverse polarity, output overvoltage, severe overcurrent/short-circuit, and internal control/sensing/gate-drive faults

All fault types disable the output while active.

In wall-assist builds, "disable the output" means opening the common load disconnect in addition to disabling buck switching. Branch reverse blocking and branch protection must continue to isolate an unpowered, disabled, or faulted source. A wall-branch fault may disable the wall contribution without unnecessarily disabling a healthy PV branch, unless the shared-bus or load protection requires a system-level shutdown.

The PV-source current measurement supports enforcement of the rated PV input-current ceiling. The controller shall constrain the delivered operating point before that ceiling is exceeded.

The MPPT-inductor current measurement supports control, telemetry, and overcurrent detection. No amplifier/comparator path shall be claimed as cycle-by-cycle protection until its complete propagation-delay and current-overshoot budget has been verified.

A precision telemetry/control amplifier and a hardware-fast fault path may be separate functions if one device cannot meet both accuracy and protection-speed requirements.

The architecture should define:
- detection source for each representative fault
- fault class assignment
- output shutdown path
- retry/clear conditions
- telemetry reporting
- last-fault retention behavior

Exact thresholds, timings, and full taxonomy remain open.

---

## 11. Thermal Architecture

The v1 architecture shall include:
- at least one thermally meaningful converter temperature measurement
- thermal telemetry
- overtemperature protection
- documented operating envelope
- documented cooling/airflow/heatsinking assumptions
- safe behavior outside allowable thermal limits

Detailed design work shall define:
- sensor placement
- thermal thresholds
- cooldown/retry behavior
- derating curves if used
- airflow or heatsinking assumptions for continuous operation throughout the rated output envelope

---

## 12. Configuration, Persistence, and Local Controls

The architecture shall provide nonvolatile storage for:
- operational configuration
- calibration data
- startup-related settings
- persisted limits and setpoints required by v1 behavior

Startup behavior:
- use active nonvolatile configuration when valid
- fall back to factory defaults when no valid active configuration exists
- validate and clamp persisted values before use
- require a valid auxiliary `PGOOD` indication before PWM can be armed
- retain HRTIM and gate-driver commands in their inactive states until startup validation completes

Factory-default behavior:
- output enabled
- regulate-only mode
- soft-start enabled
- reference/default output setting of 12.0 V unless configured otherwise

Local controls/indication:
- status LED(s)
- local reset / fault-clear control
- local enable / disable control
- factory-default-versus-modified operational configuration indicator

Calibration-only changes shall not count as operational configuration changes for the factory-default indicator.

---

## 13. Verification Architecture

The architecture shall support a verification approach that demonstrates:
- stable nominal converter operation
- representative software-configurable output voltage setpoints
- constrained source optimization
- v1 external-interface telemetry/control/logging
- supervisory-UART communication, external-host backend translation, and basic coordinated Bitaxe control through stock AxeOS
- source-limited-state behavior
- representative automatic-retry fault behavior
- representative latched fault behavior
- thermal telemetry/protection behavior
- completion of open deliverables required for reproducibility
- reverse-current blocking and fault isolation on both source branches
- PV-only, wall-only, both-source, source-order, and loss-of-source transitions
- wall-input voltage/current/power telemetry and common load-disconnect behavior

A stretch wall-assist demonstration should hold the miner online while increasing PV availability causes a measured, corresponding decrease in wall-input power.

Detailed test procedures and pass/fail criteria shall be captured in later verification/test planning artifacts.

---

## 14. Open Deliverables and Release Artifact Plan

The architecture should preserve a clean path to open release of:
- native editable schematic source
- native editable PCB layout source
- fabrication outputs
- pick-and-place / component placement file where applicable
- assembly outputs or notes
- BOM with manufacturer part numbers, key specifications, and approved alternates where practical
- firmware source
- interface documentation
- build/programming instructions
- bring-up and calibration instructions
- verification/test procedure documentation
- v1 Bitaxe coordinated-control demo procedure documentation

Generated outputs shall not be the only released form of the design artifacts.

Current public artifacts include the working KiCad project and circuit sheets,
PCB scaffold, CubeMX IOC, and README image. Components are a separate sibling
`ocpm-library` checkout pinned by `../hardware/ocpm-library.lock`; see
`../hardware/README.md` for reproduction instructions. This is development
source, not a completed v1 release. Exact licenses, manufacturing outputs,
validated firmware/toolchain instructions, and release packaging remain open.

---

## 15. Open Architecture Questions

Questions to resolve in later architecture/design passes:

- After the initial STM32G474 LQFP48 pinout/CubeMX seed check, do the detailed HRTIM, ADC trigger, comparator/op-amp, and hardware-shutdown routes remain comfortable?
- Which remaining PV-source and wall-sensing parts meet their requirements, and does the selected SEWF3920 + INA241A2 MPPT-inductor channel meet its reference, headroom, accuracy, and ripple-response targets?
- Does the independent predictive low-side OCP path meet the one-pulse bound, or is an alternative qualified fast-protection path required? The precision INA241 channel is not the assumed severe-OCP path.
- Can the final STM32G474 pinout support an optional future dual-internal-op-amp inductor-current experiment without compromising required functions?
- After the common load-disconnect/protection architecture is selected, should total load current be measured directly or derived from validated branch measurements?
- Implement and validate the PB13 / ADC3_IN5 delivered-load-voltage sensing circuit and acquisition timing; LOAD_V_SENSE naming is already reconciled in schematic/IOC, and optional TEMP_SPARE is displaced.
- Validate the placed 402 kΩ / 100 kΩ auxiliary EN divider, including EN leakage, falling-threshold uncertainty, and startup/collapse behavior.
- Validate auxiliary PG_5V0 startup/collapse handling on its selected PA15 pin.
- Which protection paths should be hardware-fast versus firmware-managed?
- What exact source-limited behavior should occur when the source cannot support the requested output?
- What MPPT algorithm should implement constrained local tracking?
- What qualifies as sufficient rail margin for source exploration?
- What exact packet framing, freshness indication, and command lease behavior should the backend-neutral supervisory UART use?
- Which AxeOS telemetry fields and writable operating parameters should the host use for the initial v1 demo?
- What additional primitives should be proposed or implemented in Mujina based on the source-aware-power experiments?
- Should a future UART-to-network gateway own policy/allocation or only protocol translation, discovery, and routing?
- What external command/status protocol is simplest and most useful for v1?
- Should the debug UART include an interactive console?
- What thermal model and cooling assumptions are appropriate for 100 W operation?
- Which open-source and open-hardware licenses should be used?
- What wall-supply nominal voltage/tolerance and MPPT bus ceiling provide adequate priority-sharing margin across current-dependent path drops?
- Which ideal-diode/reverse-blocking, wall-branch protection, and common load-disconnect devices meet the voltage, current, loss, and single-fault requirements?
- What transient envelope and output capacitance are required to ride through rapid PV loss until the wall supply assumes the deficit?

Related artifact:
- `sourcerer_stm32g474_pin_peripheral_budget.md`
