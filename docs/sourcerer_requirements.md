# Sourcerer System Requirements

**Publication scope:** This repository currently publishes the requirements and
architecture documents only. References to other working notes, plans, and
editable diagrams name unpublished project artifacts.

_Status: revised collaborative draft_

**Documentation review:** 2026-09-10 — project naming and links updated; requirement IDs, normative statements, and numeric limits unchanged.

## 1. Purpose

This document captures the **high-level requirements** for Sourcerer v1.

Its purpose is to:
- keep the project scoped
- separate requirements from implementation details
- provide a lighter-weight staged design process inspired by SDR / PDR / CDR without excessive overhead
- establish what v1 must do before deeper architecture and detailed design work proceed

This document should stay focused on:
- what v1 is supposed to achieve
- what is explicitly out of scope
- what interfaces, behaviors, and protections are required
- what constitutes success

It should avoid prematurely locking in detailed implementation choices unless they become necessary for scope control.

---

## 2. Relationship to other project docs

This document is the authoritative source for v1 requirements and the rated
electrical envelope.

Related documents have the following roles:

- `../README.md` = public, non-normative project overview
- `sourcerer_requirements.md` = what v1 must and must not do
- `sourcerer_system_architecture.md` = system partitioning, hardware and
  high-level firmware architecture, behavior, diagrams, and state machines
- `hardware_design_decisions.md` = rationale behind hardware design choices
- `../PROJECT_STATUS.md` = current implementation state, blockers, and resume point
- `sourcerer_bringup_plan.md` and `validation_plan.md` = bring-up and
  verification procedures

Legacy concept notes, educational design notes, and archived journals are
historical background rather than current design authority. Their content
becomes authoritative only when deliberately transferred into the appropriate
maintained document.

Detailed interface, protocol, and software-decision documentation may be added
as those designs mature.

---

## 3. v1 scope

### 3.1 Intended v1 deliverable

The v1 deliverable is intended to be:
- an **MPPT converter reference design / development platform**
- including converter hardware and firmware
- plus a **basic integrated miner/load demo**
- specifically enough to demonstrate **basic coordinated control with a Bitaxe board**

Important clarification:
- v1 is **not** intended to be a polished end-user product
- v1 is intended to be a technically credible and extensible reference platform
- v1 is intended to be **bench-usable and reproducible by another engineer**

### 3.2 Project framing for v1

For v1, the project is framed as:
- an **MPPT supply first**
- with **coordination hooks and a basic Bitaxe-facing demo**
- rather than a fully developed coordinated mining-power ecosystem from day one

### 3.3 Primary v1 load focus

v1 is explicitly **Bitaxe-focused** rather than being framed as a generic converter platform first.

This does not prevent broader reuse later, but v1 should stay anchored to the Bitcoin-mining use case.

### 3.4 Power-stage framing

v1 is explicitly a **non-isolated buck converter** reference design.

This means:
- boost capability is not required for v1
- behavior at low input voltage shall be handled through defined source-limited / dropout behavior rather than by requiring boost operation

### 3.5 Wall-assist framing

The first custom v1 PCB shall preserve an optional **wall-assist** path from an external, fixed-output, isolated AC/DC supply.

For scope control:
- the PCB accepts only the supply's safety-extra-low-voltage DC output; AC mains conversion is not part of this board
- the MPPT branch remains independently usable with wall assist physically disconnected
- standalone buck-converter bring-up precedes wall-assist bring-up
- wall assist is solar-first priority sharing, not equal current sharing: available PV power displaces wall power and the wall source supplies the deficit
- the PV/DC input source is dedicated to Sourcerer within the v1 system boundary

---

## 4. Explicitly out of scope for v1

The following items are explicitly out of scope for v1:
- battery integration
- battery charging
- battery management
- battery-integrated operating modes
- onboard AC mains conversion
- onboard Wi-Fi, Ethernet, or other network connectivity
- remote firmware update capability
- production-ready packaging
- polished software ecosystem
- formal EMI/EMC compliance or certification
- coordination of a PV source or DC bus shared with upstream home loads, storage, inverters, or other non-Sourcerer consumers

These may become later roadmap items, but they should not expand the scope of the first deliverable.

---

## 5. System context

The v1 system is intended to be a solar-input MPPT power converter platform that:
- accepts power from photovoltaic sources
- is designed and validated primarily around the nominal PV range defined by the rated electrical envelope
- may operate below that nominal PV range when sufficient voltage and power are available
- dynamically restricts its allowed output operating point when source voltage, source power, or converter headroom is insufficient
- remains safe when the source is de-energized or below the functional operating range
- provides a software-configurable output within the rated electrical envelope
- powers a Bitaxe-focused downstream system
- exposes digital supervisory and debug interfaces
- demonstrates basic coordinated control with a Bitaxe board

The v1 communications concept is intentionally local and wired at the
Sourcerer boundary:
- a **backend-neutral supervisory UART** connected to an external host
- a **separate debug UART** for development use

The v1 coordinated-control demonstration uses a Raspberry Pi-class external
host to translate between the Sourcerer supervisory interface and a supported
miner-control backend. The initial backend is stock AxeOS over its network API;
Mujina is an alternate and strategic future backend. Onboard Sourcerer
wireless/network control is not required for v1, and miner-specific network
APIs shall not be embedded in the converter firmware.

---

## 6. Functional requirements

### 6.1 Core functionality

**REQ-FUNC-001**  
The system shall perform as a solar-input MPPT power converter rather than as a fixed uncoordinated DC supply.

**REQ-FUNC-002**  
The system shall support powering a Bitaxe-focused downstream load in v1.

**REQ-FUNC-003**  
The system shall support a basic integrated demonstration of coordinated control with a Bitaxe board.

**REQ-FUNC-004**  
The system shall expose digital supervisory and debug interfaces suitable for telemetry, setpoints, diagnostics, and mode control.

### 6.2 Operating modes

**REQ-FUNC-005**  
The system shall support a **regulate-only** operating mode in which output regulation is maintained without active MPPT exploration.

**REQ-FUNC-006**  
The system shall support a **constrained local tracking** mode for bounded source optimization.

**REQ-FUNC-007**  
The system shall support externally selectable operating modes through the supervisory interface.

### 6.3 MPPT behavior constraints

**REQ-FUNC-008**  
The system shall support bounded/local source tracking behavior without requiring disruptive wide-range sweeps in normal operation.

---

## 7. Electrical requirements

### 7.1 Rated electrical envelope and input behavior

**REQ-ELEC-001**  
The authoritative v1 rated electrical envelope shall be:

| Parameter                           |           Value | Classification                                                           |
| ----------------------------------- | --------------: | ------------------------------------------------------------------------ |
| Nominal PV input range              |    18 V to 44 V | Design and validation target                                             |
| Maximum continuous PV input voltage |            50 V | Hard operating limit                                                     |
| PV input survival/transient ceiling |            60 V | Non-operating survival limit; allowable waveform and duration remain TBD |
| Maximum PV input current            |            10 A | Hard operating limit                                                     |
| Output-voltage setpoint range       | 5.0 V to 16.0 V | Hard command and operating range                                         |
| Maximum output current              |            10 A | Hard operating limit                                                     |
| Maximum output power                |           100 W | Hard operating limit                                                     |

This table is the authoritative source for these numeric limits. Other requirements shall reference the rated electrical envelope rather than duplicating its values.

**REQ-ELEC-002**  
The system shall target design and validation primarily within the nominal PV input range defined by the rated electrical envelope.

**REQ-ELEC-003**  
The hardware shall tolerate the input decreasing to 0 V without damage.

**REQ-ELEC-004**  
An input of 5 V shall be a safe non-operating and non-regulating condition. Startup or regulated output operation shall not be required at that input voltage.

**REQ-ELEC-005**  
The system shall constrain delivered output before PV input current exceeds the maximum defined by the rated electrical envelope.

### 7.2 Output

**REQ-ELEC-006**  
The system shall provide a **software-configurable output voltage**.

**REQ-ELEC-007**  
The commanded and allowed output-voltage setpoint shall remain within the rated electrical envelope.

**REQ-ELEC-008**  
The default output voltage shall be configurable, and the reference factory-default initial setting should be 12.0 V.

**REQ-ELEC-009**  
When sufficient source power is available, the system shall support continuous operation throughout the rated output envelope, subject to documented thermal derating.

**REQ-ELEC-010**  
All output limits in the rated electrical envelope shall apply simultaneously. The controller shall not permit an operating point that exceeds any applicable limit.

### 7.3 Efficiency

**REQ-ELEC-011**  
The v1 design should target approximately 95% efficiency under nominal operating conditions. This is a nonbinding design objective and is not a universal pass/fail requirement for v1.

### 7.4 First-PCB wall-assist electrical path (optional to populate/use)

**REQ-ELEC-012**  
The first custom v1 PCB shall include a wall-assist DC input path compatible with an external fixed-output, isolated AC/DC supply whose output is suitable for the active 12 V-class load rail.

**REQ-ELEC-013**  
The wall-assist path shall include a suitably rated, user-accessible physical disconnect such as a removable fuse/link or connector so the branch can be electrically absent during standalone MPPT bring-up and use.

**REQ-ELEC-014**  
The MPPT branch and wall-assist branch shall each prevent reverse current into an unpowered, disabled, disconnected, or faulted source over the documented operating envelope.

**REQ-ELEC-015**  
Each source branch shall include protection appropriate to its conductor, connector, source capability, and credible branch faults.

**REQ-ELEC-016**  
When both sources are available, the system shall prioritize available PV power and use wall power only to supply the remaining load deficit, subject to converter and bus limits.

**REQ-ELEC-017**  
The wall-assist voltage target, MPPT bus-voltage ceiling, path drops, tolerances, and handoff hysteresis shall be coordinated so source handoff does not produce sustained chatter or uncontrolled circulating current.

**REQ-ELEC-018**  
The system shall support safe startup, operation, and loss-of-source behavior for PV only, wall only, both sources, neither source, PV-first startup, and wall-first startup.

**REQ-ELEC-019**  
Failure, shutdown, or absence of either source branch shall not backfeed or collapse the other healthy source under the documented single-fault assumptions.

**REQ-ELEC-020**  
With wall assist connected, an output-disable command or fault requiring output shutdown shall disconnect the load from the combined source bus rather than relying only on disabling the buck gate drive.

**REQ-ELEC-021**  
The system shall measure wall-input voltage and wall-input current and shall expose derived wall-input power telemetry sufficient to observe source contribution and verify solar-first behavior.

**REQ-ELEC-022**  
The wall-input measurements should target approximately **±2% voltage accuracy** and **±5% current accuracy** under calibrated conditions.

**REQ-ELEC-023**  
With a suitable wall supply present and within the documented transient envelope, rapid loss of PV contribution shall not drive the load rail below the active output-undervoltage threshold.

**REQ-ELEC-024**  
Wall-assist hardware shall not be required for standalone MPPT operation, calibration, or the initial converter bring-up sequence.

**REQ-ELEC-025**  
The wall-assist branch shall provide controller-commanded electrical isolation in addition to the user-accessible physical disconnect and automatic reverse-current blocking.

**REQ-ELEC-026**  
Wall assist shall be enabled only when the detected wall-source voltage is compatible with the commanded load-rail setpoint and documented handoff margin; otherwise the controller shall isolate the wall branch, keep the common load disconnect open until the rail is safe, and report the incompatible-source condition.

---

## 8. Control and safety requirements

### 8.1 Output stability behavior

**REQ-SAFE-001**  
The system shall maintain stable and controlled output-rail behavior during regulation and MPPT operation.

### 8.2 Buck-oriented behavior / low-input handling

Because v1 is a non-isolated buck design, there will be operating conditions in which input voltage is insufficient to maintain the commanded output.

**REQ-SAFE-002**  
When the input source cannot support the commanded operating point within valid operating limits, the system shall enter a defined source-limited operating state rather than operating unpredictably.

**REQ-SAFE-003**  
The source-limited operating state shall be indicated in the telemetry data.

### 8.3 Output undervoltage shutdown behavior

**REQ-SAFE-004**  
The system shall support a configurable output undervoltage shutdown margin referenced to the commanded output voltage.

**REQ-SAFE-005**  
The default output undervoltage shutdown margin shall be **0.5 V**.

**REQ-SAFE-006**  
The output undervoltage shutdown margin shall be supervisor-configurable.

**REQ-SAFE-007**  
The output undervoltage shutdown margin shall be greater than zero.

**REQ-SAFE-008**  
The output undervoltage shutdown threshold shall be derived from the commanded output voltage minus the active output undervoltage shutdown margin.

**REQ-SAFE-009**  
The derived output undervoltage shutdown threshold shall not be less than **4.5 V**.

**REQ-SAFE-010**  
If output voltage drops below the active output undervoltage shutdown threshold, the system shall disable output immediately.

**REQ-SAFE-011**  
The telemetry data shall indicate when output has been disabled due to violation of the active output undervoltage shutdown threshold.

### 8.4 Protection philosophy

**REQ-SAFE-012**  
The auxiliary regulator shall use an external resistor divider on its enable input targeting nominal `+AUX_IN` thresholds of 6.2 V rising and 5.6 V falling. Guaranteed threshold bounds shall be documented after accounting for regulator enable-threshold and resistor tolerances.

**REQ-SAFE-013**  
The system shall include input overvoltage protection or limiting behavior appropriate to safe operation.

**REQ-SAFE-014**  
The system shall include input-current protection or limiting behavior that prevents PV input current from exceeding the maximum defined by the rated electrical envelope.

**REQ-SAFE-015**  
The system shall include input reverse-polarity / reverse-voltage protection appropriate to safe operation.

**REQ-SAFE-016**  
The system shall include output overcurrent protection and/or current limiting behavior appropriate to safe operation.

**REQ-SAFE-017**  
The system shall include output overvoltage protection behavior appropriate to safe operation.

**REQ-SAFE-018**  
The system shall detect output undervoltage relative to allowed operating limits.

**REQ-SAFE-019**  
The system shall include overtemperature protection behavior appropriate to safe operation.

Note:
- exact thresholds, trip behavior, and recovery behavior for these protections will be refined in later architecture/design documents

### 8.5 Fault behavior

**REQ-SAFE-020**  
The fault-handling strategy for v1 shall support the **automatic-retry fault class**.

**REQ-SAFE-021**  
Faults assigned to the automatic-retry fault class shall clear automatically when the condition causing the fault is no longer present.

**REQ-SAFE-022**  
The fault-handling strategy for v1 shall support the **latched fault class**.

**REQ-SAFE-023**  
Faults assigned to the latched fault class shall remain asserted until cleared by their defined recovery action.

**REQ-SAFE-024**  
All fault types shall disable the output while the fault is active.

**REQ-SAFE-025**  
The **insufficient-source fault type** shall belong to the automatic-retry fault class.

**REQ-SAFE-026**  
The **input undervoltage fault type** shall belong to the automatic-retry fault class.

**REQ-SAFE-027**  
The **output undervoltage due to source collapse fault type** shall belong to the automatic-retry fault class.

**REQ-SAFE-028**  
The **overtemperature fault type** shall belong to the automatic-retry fault class, with cooldown as part of its recovery condition.

**REQ-SAFE-029**  
The **input overvoltage fault type** shall belong to the automatic-retry fault class, with return to the valid operating range as part of its recovery condition.

**REQ-SAFE-030**  
The **reverse-polarity fault type** shall belong to the latched fault class and shall remain asserted until the fault condition is removed and the system is power-cycled.

**REQ-SAFE-031**  
The **output overvoltage fault type** shall belong to the latched fault class and shall remain asserted until cleared by supervisory command or power cycle.

**REQ-SAFE-032**  
The **severe output overcurrent or short-circuit fault type** shall belong to the latched fault class and shall remain asserted until cleared by supervisory command or power cycle.

**REQ-SAFE-033**  
The **internal control, sensing, or gate-drive fault type** shall belong to the latched fault class and shall remain asserted until cleared by supervisory command or power cycle.

**REQ-SAFE-034**  
The telemetry data shall indicate the active fault class and fault type.

**REQ-SAFE-035**  
The system shall retain the last asserted fault class and fault type for diagnostic purposes.

Note:
- exact numeric input-side thresholds and detailed fault taxonomy will be refined in later architecture/design documents

---

## 9. Interface and telemetry requirements

### 9.1 Control and interface surfaces

**REQ-INTF-001**  
The system shall provide a backend-neutral supervisory UART for bidirectional telemetry, status, configuration, and operating-command exchange with an external host.

**REQ-INTF-002**  
The supervisory UART shall expose the Sourcerer telemetry and status required for the v1 coordinated-control demo without requiring knowledge of AxeOS, Mujina, or another miner-specific API in the converter firmware.

**REQ-INTF-003**  
The supervisory UART shall accept the bounded Sourcerer operating commands required for the v1 coordinated-control demo.

**REQ-INTF-004**  
The system shall include a documented backend-neutral command/status interface for the supervisory UART connection.

**REQ-INTF-005**  
The system shall provide a separate UART interface for debug/development use.

**REQ-INTF-006**  
The debug UART shall support logging and diagnostics.

**REQ-INTF-007**  
The reference design shall provide bench-accessible, documented UART, programming, and debug access.

### 9.2 Minimum command/status capability set

**REQ-INTF-008**  
The system shall expose a documented minimum command/status capability set across its v1 external interfaces, even though exact packet/protocol definitions are left to later interface documentation.

**REQ-INTF-009**  
The documented minimum read/status capability set shall include at least:
- input voltage
- input current
- input power
- output voltage
- output current
- output power
- at least one meaningful converter temperature
- fault status
- current operating mode/state
- source-limited status
- hardware revision identification
- firmware version identification
- supervisory-host connection/status sufficient for the v1 coordinated-control demo
- wall-input voltage, current, power, presence/status, and load-disconnect status when wall-assist hardware is populated

**REQ-INTF-010**  
The documented minimum control capability set shall include at least:
- set output voltage
- set output undervoltage shutdown margin
- set output current limit
- set output power limit
- select operating mode
- enable output
- disable output
- clear/reset fault

### 9.3 Mandatory telemetry for v1

**REQ-INTF-011**  
The system shall expose input-side telemetry sufficient to observe source condition and source power.

**REQ-INTF-012**  
The system shall expose output-side telemetry sufficient to observe rail condition and delivered load power.

**REQ-INTF-013**  
When wall assist is populated, the system shall expose source-contribution telemetry sufficient to distinguish PV contribution from wall contribution.

**REQ-INTF-014**  
The system shall expose temperature and fault telemetry sufficient for development, debugging, and safe supervisory control.

**REQ-INTF-015**  
The system shall expose current mode/state and source-limited status.

**REQ-INTF-016**  
Externally exposed telemetry shall support a useful real-time update rate, with a working v1 target of approximately **5 Hz to 10 Hz** for key telemetry and status reporting.

**REQ-INTF-017**  
Internal sensing/control update rates are intentionally left to architecture and are not fixed in this requirements revision.

### 9.4 Measurement and calibration

**Acquisition clarification (2026-09-10):** PV voltage and current do not
need simultaneous sampling. Sequential acquisition is permitted, provided
conditioning, timing, and averaging support the specified calibrated
measurement and power-telemetry objectives. This does not remove the separate
architecture's fast local voltage/inductor-current simultaneous-pair intent.

**REQ-INTF-018**  
The reference design shall provide developer/factory calibration capability for key measured quantities used in telemetry, control, protection, and validation.

**REQ-INTF-019**  
The v1 design should target the following approximate measurement accuracies under calibrated conditions:
- input voltage: **±2%**
- output voltage: **±1%**
- input current: **±5%**
- output current: **±5%**

**REQ-INTF-020**  
Power telemetry shall be derived from calibrated measurements.

**REQ-INTF-021**  
Power telemetry shall be sufficient for MPPT operation, debugging, and validation.

**REQ-INTF-022**  
Temperature measurement accuracy is not given a hard numeric requirement in this revision, but shall be sufficient for meaningful protection and debugging.

### 9.5 Logging and firmware update constraints

**REQ-INTF-023**  
The system shall support clean host-side logging of telemetry/status via its external interfaces.

**REQ-INTF-024**  
Onboard logging/history storage is not required for v1.

**REQ-INTF-025**  
Firmware programming and update capability for v1 shall be local-only.

---

## 10. Configuration, persistence, and local indication requirements

### 10.1 Nonvolatile configuration and persistence

**REQ-CFG-001**  
The system shall provide nonvolatile storage for configuration, calibration data, startup-related settings, and other persisted operational parameters required by v1 behavior.

**REQ-CFG-002**  
The system shall provide factory-default configuration settings.

**REQ-CFG-003**  
At startup, the system shall initialize according to the active non-volatile configuration settings, or factory-default configuration settings if no valid active configuration is available.

**REQ-CFG-004**  
The factory-default startup configuration shall enable output.

**REQ-CFG-005**  
The factory-default startup configuration shall use regulate-only mode.

**REQ-CFG-006**  
The system shall provide a configuration setting for enabling or disabling soft-start.

**REQ-CFG-007**  
The factory-default configuration shall have soft-start enabled.

**REQ-CFG-008**  
The output voltage setpoint may persist across power cycle.

**REQ-CFG-009**  
Persisted output voltage setpoints shall be validated and clamped to safe allowed bounds before use at startup.

**REQ-CFG-010**  
Configured current and power limits may persist across power cycle.

**REQ-CFG-011**  
Persisted current and power limits shall be validated and clamped to safe allowed bounds before use at startup.

**REQ-CFG-012**  
The output undervoltage shutdown margin may persist across power cycle.

**REQ-CFG-013**  
Persisted output undervoltage shutdown margin settings shall be validated and clamped to safe allowed bounds before use at startup.

### 10.2 Factory-default configuration awareness

**REQ-CFG-014**  
The system shall maintain a notion of **factory-default operational configuration**.

**REQ-CFG-015**  
The system shall determine whether the current active operational configuration differs from factory-default operational configuration.

**REQ-CFG-016**  
A local indicator shall be asserted whenever the active operational configuration differs from factory-default operational configuration.

**REQ-CFG-017**  
The factory-default-versus-modified indication shall be available both at startup and during normal operation.

**REQ-CFG-018**  
Calibration-only changes shall not count as operational configuration changes for the purpose of the factory-default indicator.

### 10.3 Local interface and debug access

**REQ-CFG-019**  
The system shall provide local status indication via status LED(s).

**REQ-CFG-020**  
The system shall provide a local reset / fault-clear control.

**REQ-CFG-021**  
The system shall provide a local enable / disable control.

**REQ-CFG-022**  
The reference design shall provide bench-friendly firmware programming and debug access.

**REQ-CFG-023**  
A runtime command that would set output voltage, output current limit, or output power limit outside the applicable rated electrical envelope shall be rejected atomically, and the previous valid setting shall be retained.

**REQ-CFG-024**  
When a runtime command is rejected, the system shall report the rejection and identify the invalid setting or applicable limit.

---

## 11. Bitaxe integration requirements

**REQ-BITAXE-001**  
The system shall not require upstream Bitaxe firmware changes in order to achieve safe basic supply operation.

**REQ-BITAXE-002**  
If the external host is absent, its command is stale or invalid, or miner identity, telemetry, firmware behavior, writable-control support, or firmware settings are not known to be safe for coordinated operation, the converter shall fall back to conservative supply behavior and shall not rely on host-side or miner-side cooperation for safe operation.

**REQ-BITAXE-003**  
The v1 system shall be single-load focused, while leaving room for future extension to broader coordination architectures.

**REQ-BITAXE-004**  
The v1 system shall support **basic coordinated control** with a Bitaxe board, not merely communications exchange.

**REQ-BITAXE-005**  
The coordinated Bitaxe demo shall use an external host to control at least one power-relevant writable Bitaxe operating parameter through a supported miner-control backend.

**REQ-BITAXE-006**  
The commanded Bitaxe operating parameter shall produce an observable load-relevant effect sufficient to demonstrate basic power coordination.

**REQ-BITAXE-007**  
The exact required Bitaxe operating parameter is intentionally left open in this revision, but mining enable/disable is acceptable as a minimum viable control target and frequency setpoint is preferred if available and safe.

**REQ-BITAXE-008**  
The initial v1 miner-control backend shall support a Bitaxe running stock AxeOS through the AxeOS network API without requiring a direct Sourcerer-to-Bitaxe communications connection.

**REQ-BITAXE-009**  
The external-host implementation shall keep miner-control backends modular so the Sourcerer supervisory interface can be used with AxeOS or Mujina without changing the converter firmware protocol.

**REQ-BITAXE-010**  
Mujina integration is a strategic development target, but shall not be required to satisfy the initial v1 coordinated-control acceptance demonstration.

**REQ-BITAXE-011**  
The responsibility split of a possible future UART-to-Wi-Fi/Ethernet gateway, including whether it performs policy/allocation or only protocol translation and routing, is intentionally not fixed by the v1 requirements.

---

## 12. Environmental and implementation requirements

**REQ-IMPL-001**  
The v1 operating ambient temperature target shall be **-20°C to 50°C**.

**REQ-IMPL-002**  
The v1 storage / non-operating ambient temperature target shall be **-40°C to 85°C**.

**REQ-IMPL-003**  
The v1 design shall use EMI/EMC-aware power, filtering, grounding, and PCB-layout practices, including practical mitigation and tuning provisions appropriate for a switching power reference design, such as input/output filtering provisions, controlled high-current loop layout, grounding strategy, and damping/snubber options where appropriate.

**REQ-IMPL-004**  
Formal EMI/EMC compliance/certification is not required for v1.

**REQ-IMPL-005**  
The v1 thermal operating envelope, including any required airflow, heatsinking, derating, or other cooling assumptions, shall be documented.

**REQ-IMPL-006**  
The physical design shall support safe thermal behavior within the documented operating envelope and shall fail safely outside allowable thermal limits.

**REQ-IMPL-007**  
Temperature sensing used for telemetry and protection shall correspond to at least one thermally meaningful converter location.

**REQ-IMPL-008**  
The reference design shall provide bench-usable access for:
- power input/output
- supervisory-host UART
- debug UART
- programming/debug access

**REQ-IMPL-009**  
Bench-accessible power input/output connections shall be appropriate for the v1 voltage and current design points.

---

## 13. Open-source, documentation, and reproducibility requirements

**REQ-OPEN-001**  
The v1 project shall provide open firmware/source code under an open-source license.

**REQ-OPEN-002**  
The v1 project shall provide open hardware design files under an open hardware license.

**REQ-OPEN-003**  
The v1 project shall provide open documentation sufficient for another engineer to understand, build, bring up, test, and reproduce the reference design.

**REQ-OPEN-004**  
The open deliverables shall include at least:
- native editable schematic source
- native editable PCB layout source
- fabrication outputs suitable for PCB manufacture
- pick-and-place / component placement file where applicable
- assembly outputs or notes suitable for bench-scale assembly
- BOM with manufacturer part numbers, key component specifications, and approved alternates where practical
- firmware source
- interface documentation
- build/programming instructions
- bring-up and calibration instructions
- verification/test procedure documentation
- v1 Bitaxe coordinated-control demo procedure documentation

**REQ-OPEN-005**  
Generated outputs such as PDFs, images, fabrication files, or documentation exports shall not be the only released form of the hardware or firmware design artifacts.

---

## 14. Verification / success criteria

### 14.1 High-level success definition

At a high level, v1 is successful if it delivers:
- a credible MPPT converter reference design / dev platform
- safe output behavior for a Bitaxe-focused load
- software-configurable output centered on a 12 V-class rail
- constrained source-optimization behavior
- telemetry, setpoint, and mode-control hooks
- a basic coordinated-control demonstration with a Bitaxe board
- open and reproducible design artifacts sufficient for another engineer to build and evaluate the reference design

### 14.2 Draft v1 success criteria

**REQ-VER-001**  
The v1 platform shall demonstrate stable controlled operation as a solar-input MPPT converter in a bench setting appropriate for development and validation.

**REQ-VER-002**  
The v1 platform shall demonstrate a Bitaxe-focused integrated demo rather than a converter-only bench artifact.

**REQ-VER-003**  
The v1 platform shall demonstrate end-to-end basic coordinated control with a Bitaxe board through the backend-neutral Sourcerer supervisory interface, an external host, and a supported miner-control backend, including commanding at least one power-relevant Bitaxe operating parameter with an observable load-relevant effect.

**REQ-VER-004**  
The v1 platform shall demonstrate safe behavior when source conditions limit the ability to maintain the commanded operating point.

**REQ-VER-005**  
The v1 project shall demonstrate completion of the open deliverables required for reproducibility.

### 14.3 Required demo / verification scenarios

**REQ-VER-006**  
The v1 verification approach shall include demonstration of stable nominal converter operation.

**REQ-VER-007**  
The v1 verification approach shall include demonstration of software-configurable output voltage operation at representative v1 setpoints, including nominal 12 V-class operation and lower-voltage Bitaxe-support operation.

**REQ-VER-008**  
The v1 verification approach shall include demonstration of constrained source-optimization behavior.

**REQ-VER-009**  
The v1 verification approach shall include demonstration of v1 external-interface telemetry, control, and host-side logging behavior.

**REQ-VER-010**  
The v1 verification approach shall include demonstration of the Sourcerer supervisory UART, external-host backend translation, and basic coordinated Bitaxe control through stock AxeOS, including reading the telemetry needed by the host and commanding at least one power-relevant Bitaxe operating parameter with an observable load-relevant effect.

**REQ-VER-011**  
The v1 verification approach shall include demonstration of source-limited-state behavior.

**REQ-VER-012**  
The v1 verification approach shall include demonstration of representative automatic-retry fault behavior.

**REQ-VER-013**  
The v1 verification approach shall include demonstration of representative latched fault behavior.

**REQ-VER-014**  
The v1 verification approach shall include demonstration of thermal telemetry and protection behavior sufficient to support the documented v1 thermal operating envelope.

**REQ-VER-015**  
The v1 verification approach shall confirm reverse-current blocking and fault isolation for the PV and wall-assist branches, including an unpowered source on either branch.

**REQ-VER-016**  
The v1 verification approach shall exercise PV-only, wall-only, both-source, PV-first, wall-first, and loss-of-either-source transitions.

**REQ-VER-017**  
As a stretch verification scenario, the system should maintain uninterrupted mining while increasing available PV power produces a corresponding measured decrease in wall-input power.

---

## 15. Items intentionally left open for later architecture/design work

The following items are intentionally not fully fixed in this requirements revision:
- remaining protection thresholds, trip behavior, recovery behavior, and timing not explicitly fixed by this requirements document
- detailed fault taxonomy beyond the v1 fault classes and representative fault types already captured
- exact command packet/protocol definitions
- exact backend-neutral supervisory-UART packet framing and command mapping
- exact AxeOS and Mujina host-backend implementation details
- exact power-relevant Bitaxe operating parameter used for the v1 coordinated-control demo
- whether a future UART-to-network gateway owns policy/allocation or only protocol translation and routing
- exact internal sensing/control timing
- exact MPPT control algorithm
- exact realization of source-limited control response
- exact power-stage component choices, switching frequency, gate-drive architecture, and sensing topology
- implementation choices such as synchronous rectification, unless later architecture work requires them to be explicit
- exact thermal-envelope details, derating curves, airflow/heatsinking assumptions, and thermal trip thresholds
- exact open-source / open-hardware license selections
- exact manufacturing artifact formats, toolchain versions, and release packaging details
- whether interactive console access is provided on the debug UART
- exact wall-assist source rating, connector, fuse/link, ideal-diode/reverse-blocking implementation, load-disconnect device, handoff margin, and transient envelope

These items are not fixed by this requirements revision. Some implementation
choices have since been selected in `sourcerer_system_architecture.md` and
`hardware_design_decisions.md` (including synchronous buck topology, the
400 kHz prototype baseline, and the MPPT-inductor precision pair). This list
is not a claim that those project decisions are still unmade.

---

## 16. Requirements maintenance

Current project state, completed work, and the next recommended action are tracked in `../PROJECT_STATUS.md` rather than duplicated in this requirements document.
