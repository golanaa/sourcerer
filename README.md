# Sourcerer

Sourcerer is a digitally controlled solar MPPT power source with miner-aware
power budgeting and an optional wall-assist path.

Sourcerer is being developed as an open reference platform for solar-powered
Bitcoin mining. It prioritizes safe power-supply behavior while allowing the
converter and a connected Bitaxe to coordinate around available solar power.

An optional isolated wall-supply path can maintain load operation when solar
power is insufficient, while available solar power displaces wall power.

![Sourcerer v1 system overview showing power paths, local control, and miner coordination](assets/sourcerer-system-overview.png?v=fd97dd8)

## Backend-Neutral Miner Integration

Sourcerer uses a wired, backend-neutral supervisory interface rather than
embedding miner-specific protocols in the converter firmware. Initial
development connects Sourcerer over UART to a Raspberry Pi-class host, which
can interface with either:

- Bitaxes running stock AxeOS through the AxeOS network API.
- Miners managed by Mujina through the Mujina API.

This separation keeps converter regulation and protection local while allowing
the host-side coordination software to evolve independently. It also preserves
a path to a future UART-to-Wi-Fi/Ethernet gateway capable of communicating with
multiple AxeOS or Mujina instances.

## v1 Goals

- Convert available solar power into a regulated supply for a Bitaxe.
- Keep converter safety and regulation independent of external communications.
- Allow an optional isolated wall supply to fill a solar-power shortfall.
- Report power-system telemetry and demonstrate basic coordinated control with a connected Bitaxe.
- Provide an open development platform for experimenting with source-aware Bitcoin mining.
