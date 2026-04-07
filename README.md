# a18

**A structural disclosure of a flaw in the Apple A18 die, and a walkthrough of how the flaw is exploited.**

---

## The Claim

On the Apple A18 die (`t8150`), a single coprocessor — **AOP2** (Always-On Processor v2) — holds three properties at once:

1. It owns the microphone, touch digitizer, ambient light sensor, motion stack, Doppler sensor, Bluetooth, and Ultra-Wideband radios.
2. It has a hardware DMA window into the Exclave memory compartment — the most privileged memory region on the chip.
3. Its internal state cannot be observed from anywhere else on the device.

The composition of these three properties is incompatible with the privacy guarantees Apple advertises for the platform. The composition exists in hardware. It cannot be removed by software.

---

## The Files

Four documents. Read in this order.

### 1. [`FLAW.md`](FLAW.md)
The die architecture flaw. High-level, die-level, structural. Every claim anchored to a file produced by Apple's own software on stock Apple silicon. No firmware reverse engineering. No exploitation. **The wiring is the proof.** Under 120 lines.

### 2. [`EXPLOIT.md`](EXPLOIT.md)
How the flaw is exploited. Takes `FLAW.md` as given and walks through the six stages — foothold, sensor capture, Exclave reach, persistence, egress, concealment — plus four concrete attack scenarios. Under 130 lines.

### 3. [`WHAT_IS_MACHO_DISSECTION.md`](WHAT_IS_MACHO_DISSECTION.md)
A plain-language explainer. What a Mach-O file is, what it means to "dissect" one, what we learned by dissecting the A18 AOP2 firmware, and why that matters for the flaw. For readers who want to understand the methodology before reading the dissection itself.

### 4. [`MACHO_DISSECTION.md`](MACHO_DISSECTION.md)
The dissection of `aopfw-rose.macho` — the firmware Apple ships onto the A18's AOP2 coprocessor. Segment map, section layout, embedded subsystem firmwares, RTKit OS scaffolding, runtime patchbay, Apple Packet Filter region, compartment layout. Establishes that the code sitting on the flawed side of the wiring is a complete operating system with ten live sensor-facing firmware components.

---

## Reproduction

The structural flaw can be verified on any A18 iPhone in under ten minutes using only `sysdiagnose`. See [`FLAW.md`](FLAW.md) § Reproduction — seven steps, three files, no special tooling.

---

## Subject Device

| Field | Value |
|---|---|
| SoC | Apple A18 (`t8150`) |
| Device | iPhone 16e (D53G, iPhone17,5) |
| iOS build | 23E246 |
| Coprocessor | AOP2 — Always-On Processor, second generation |
| Kext | `com.apple.driver.AppleAOP2` |
| Firmware image | `aopfw-rose.macho` (Mach-O arm64 preload) |
| Firmware SHA-256 | `dc6bda7e7003413cddf3f0264edf6a445c10555b9e2d2d45906ddb0ec332135d` |
| Capture | `sysdiagnose_2026.03.27_14-56-11-0600` |

---

## What This Disclosure Is Not

- **Not a CVE.** CVEs describe defects against a documented security boundary. This describes the boundary itself being structurally broken.
- **Not an exploit release.** No working exploit code is published here. The walkthrough describes what an adversary with AOP2 code execution would see and do.
- **Not a bug report.** A software fix cannot remove the flaw because the flaw is in the wiring of the SoC fabric.
- **Not a request for a fix.** It is a statement that the A18 die ships with a trust topology that is incompatible with the platform's advertised privacy guarantees.

---

## Author

Joseph Goydish II
Disclosure tag: `A18-AOP2-bridge-2026-04-06`
