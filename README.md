# The Always-On Blind Spot

### *A die-level study of AOP2 — the Always-On Processor on the Apple A18. It owns the microphone, the touch digitizer, and the radios. It has a hardware DMA window into the most privileged memory compartment on the chip. And nothing else on the die is positioned to watch it.*

---

## The Short Version

The **Apple A18** (`t8150`) is a system-on-chip. Inside it are several independent processors that share the die: the main application processor cluster (AP), the Secure Enclave Processor (SEP), the Secure Processing Unit (SPU), and a small always-on coprocessor called **AOP2** — the Always-On Processor, second generation.

AOP2 is not a peripheral. It is a slave processor in its own right, published to the AP through the `RTBuddy` harness as `com.apple.driver.AppleAOP2`. At the silicon level, it is the owner of the microphone, the touch digitizer, the ambient light sensor, the motion stack, the Doppler sensor, Bluetooth, and Ultra-Wideband. It has its own IOMMU (`dart-aop@FC0000`) with two DMA mapper nubs — one standard, and one named `mapper-exclave-aop@1` that opens a hardware DMA window into the **Exclave** memory compartment, the hypervisor-enforced memory domain that holds biometric state, protected audio buffers, and SEP-brokered material.

AOP2 runs its own firmware. Apple's codename for it, baked into the binary, is **Rose**. Rose is a complete real-time operating system. It has its own boot stub, its own page tables, its own heap, its own runtime patch table, its own Apple Packet Filter VM, its own log compartment, and ten separate sensor-handling programs bundled inside it. It boots from Apple's preload image at SoC power-on, before the AP is running iOS, and it stays resident as long as the chip has power. When the AP wants to know what AOP2 is doing, the AP sends a message through one of 24 opaque queues (`AOP2Endpoint1..AOP2Endpoint24`) — and Rose answers. There is no second observer on the die.

The AP is wearing rose-colored glasses. It has been wearing them since the chip powered on. It can only see what Rose shows it.

---

## The Problem

This is a property of the A18 silicon, established at tape-out. **It lives in the wiring of the die, not in any line of code running on it.** The device tree... the SoC's own self-description, enumerated from `arm-io,t8150` at boot — contains this line:

```
mapper-exclave-aop@1   <IODARTMapperNub>
```

That line describes a DMA window from AOP2 into the Exclave memory compartment. It is part of the silicon fabric. The OS does not create it; the OS *discovers* it at enumeration time.

Every monitoring surface the platform exposes is downstream of AOP2 and depends on AOP2 to report honestly. The AP-side audio session lifecycle log. The privacy indicator the OS lights when an application opens a microphone. The permission-mediation database. The firehose log stream. All of them see what Rose tells them, because Rose is upstream of the pipeline every one of those observers taps into.

Every privacy property the platform advertises for A18 silicon reduces to one unwitnessable assumption: **Rose is honest, and we know this because Rose says so.** A trust model that depends on a witness vouching for itself is not a trust model.

The exploitation surface — how this topology becomes an attack in practice — is walked through in [`EXPLOIT.md`](EXPLOIT.md).

---

## The Silicon Under Study

| Field | Value |
|---|---|
| SoC | Apple A18 |
| Silicon identifier | `t8150` |
| Fabric | `arm-io,t8150` |
| Coprocessor | AOP2 — Always-On Processor, second generation |
| Coprocessor harness | `RTBuddy(AOP2)` `IOSlaveProcessor` |
| Coprocessor kext | `com.apple.driver.AppleAOP2` |
| Coprocessor IOMMU | `dart-aop@FC0000` with mapper nubs `mapper-aop@0` and `mapper-exclave-aop@1` |
| Coprocessor firmware | `aopfw-rose.macho`  codename **Rose** |
| Firmware SHA-256 | `dc6bda7e7003413cddf3f0264edf6a445c10555b9e2d2d45906ddb0ec332135d` |
| Firmware type | Mach-O 64-bit arm64 preload executable, `NOUNDEFS` |
| Adjacent silicon | SEP (Secure Enclave Processor), SPU (Secure Processing Unit), Exclave compartment |

---

## The Files

Read them in order. Under fifteen minutes to the full picture.

### 📄 [`FLAW.md`](FLAW.md) — *The die architecture flaw*
The proof, anchored entirely in the A18's own hardware enumeration. Three structural properties — sensor sovereignty, privileged DMA reach into the Exclave, and silicon-level opacity — composed on the same die. No reverse engineering, no exploitation, no special tools. **The wiring is the argument.**

### 📄 [`EXPLOIT.md`](EXPLOIT.md) — *How Rose gets weaponized*
Six stages from foothold to covert egress, plus four concrete scenarios. A table of every platform-provided detection surface and why none of them can see any of it. Takes `FLAW.md` as given and walks through what the wiring enables.

### 📄 [`WHAT_IS_MACHO_DISSECTION.md`](WHAT_IS_MACHO_DISSECTION.md) — *Plain-language methodology*
What a Mach-O file is, what it means to "dissect" one, and why we bothered. Written so a reader with no forensics background can follow the methodology before reading the dissection itself.

### 📄 [`MACHO_DISSECTION.md`](MACHO_DISSECTION.md) — *Opening up Rose*
The anatomy of `aopfw-rose.macho`: segment and section map, RTKit operating-system scaffolding, runtime sticker table, Apple Packet Filter region, compartment layout, and ten embedded sensor firmwares including the always-on voice trigger, the touch digitizer, the Doppler sensor, and the ambient light stack. Establishes that the code sitting on the flawed side of the wiring is large, capable, and exactly as structurally privileged as the wiring allows it to be.

### 📄 [`PATCHBAY.md`](PATCHBAY.md) — *The sticker table, enumerated*
A complete walk of Rose's `_rtk_patchbay` section — 62 entries, 879 bytes, zero leftover. Establishes that Rose's behavior (log level, debug state, dev mode, runtime layout, boot gating) is a set of slots the loader fills in at boot, not values fixed at build time. Full per-entry dump in [`evidence_patchbay.txt`](evidence_patchbay.txt).

---

## Reproducing The Claims

Every structural claim in [`FLAW.md`](FLAW.md) is anchored to one of three files that Apple-produced tooling generates on stock A18 silicon:

- `ioreg/IOService.txt` — the IOKit registry, as dumped by Apple's own `ioreg` utility.
- `ioreg/IODeviceTree.txt` — IOKit diagnostics, class instance counts.
- `logs/AFK/AOP2.plist` — the AFK (Apple Firmware Kit) service tree, serialized at diagnostic-capture time.

All three are produced by Apple software running on Apple silicon, with no third-party instrumentation. All three appear on any system running A18 silicon. See [`FLAW.md`](FLAW.md) § Reproduction for the exact search steps.

---

## What This Is Not

- **Not a CVE.** CVEs describe defects against a documented security boundary. This describes the boundary itself being structurally broken.
- **Not an exploit release.** No working exploit code is published in this repository. The walkthrough describes the surface the wiring exposes; it does not ship a payload.
- **Not a bug report.** You cannot file a bug against a wire in a fabric that has already been taped out.
- **Not a request for a patch.** The subject is a hardware-level property of the silicon established at tape-out, not a defect in a codebase.

It is a structural claim about the A18 die: that the composition of sensor ownership, Exclave DMA reach, and coprocessor opacity... all wired at the silicon level, all visible in the device tree, all unreachable from any observer the die provides is incompatible with the privacy properties the platform advertises. The evidence is in files the platform itself produces. 
