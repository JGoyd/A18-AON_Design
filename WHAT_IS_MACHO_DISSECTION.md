# What "Mach-O Dissection" Actually Means

A plain-language explainer for readers who want to understand what [`MACHO_DISSECTION.md`](MACHO_DISSECTION.md) is doing and why it matters.

---

## What is a Mach-O file?

**Mach-O** ("Mach object") is the file format Apple uses for all executable code on its platforms. Every app on macOS and iOS, every kernel extension, every shared library, and every piece of firmware Apple ships is a Mach-O file. It is to Apple what `.exe` / PE is to Windows and what ELF is to Linux.

A Mach-O file is not just raw machine code. It is a structured container that tells the system:

- Where the code lives inside the file
- Where the code should be loaded in memory
- What memory should be readable, writable, and executable
- What the entry point is (where execution begins)
- What external symbols it needs
- What metadata the loader should use

A program that knows how to parse the Mach-O header can walk that structure without running the code. You do not need to execute the file to understand what it is and how it is organized. You just need to read the format.

---

## What is firmware?

**Firmware** is software that runs on a chip other than the main CPU. Modern phones contain dozens of small processors — for audio, for the camera, for the modem, for the touch screen, for the Secure Enclave, and, on the Apple A18, for the Always-On Processor. Each of those processors runs its own code, loaded at power-on. That code is the firmware.

Firmware is usually invisible to the operating system's user-facing surface. It does not appear in process lists. It is not something the OS exposes for inspection. It lives on the chip, runs under the chip's own control, and exchanges messages with the main application processor (AP) through hardware mailboxes.

When Apple ships a system image for A18-class silicon, the firmware for each on-die coprocessor is embedded inside the main OS bundle. At boot, the loader places each coprocessor's firmware onto the appropriate silicon block. The file examined in this repository — `aopfw-rose.macho` — is the firmware that Apple loads onto the A18's **AOP2** (Always-On Processor, second generation).

---

## What does it mean to "dissect" a Mach-O?

Dissecting a Mach-O means parsing its structure and enumerating what is inside, the same way a biologist dissects an organism to map its anatomy. You are not running the code. You are not exploiting it. You are *reading the container* to see:

1. **What kind of file it is.** Every Mach-O begins with a "magic number" — a specific four-byte value that identifies the file as Mach-O and tells you whether it is 32-bit or 64-bit. You confirm the file is what it claims to be.

2. **What kind of program it is.** The header declares the file type — is it a standard executable, a library, a kernel extension, a preloaded firmware image? The A18 AOP2 firmware declares itself as `MH_PRELOAD`, meaning it is loaded at a fixed address by a bootloader rather than by the dynamic linker that handles normal apps. This alone tells you a great deal about how and when it runs.

3. **What segments and sections it contains.** A Mach-O is divided into **segments** (large regions, each with their own memory permissions) and, within each segment, **sections** (named sub-regions with specific purposes). Segment and section names are meaningful. `__TEXT` holds executable code. `__DATA` holds writable data. Custom names like `_rtk_boot` or `_rtk_page_tables` or `__gxf_data` reveal what Apple's engineers were building and how they named it.

4. **What strings are embedded.** Compiled code often contains string literals — error messages, log format strings, project names, build banners. Extracting the printable strings from a Mach-O gives you, essentially, a vocabulary list of what the code was designed to do. If the strings mention "voice trigger," "touch," "Doppler," or "Bluetooth," the code handles those things.

5. **What the load commands say.** A Mach-O header is followed by a series of load commands that describe how the file should be prepared for execution — where the symbol table is, where the entry point is, whether there is a code signature, and so on. Each load command is a labeled record you can enumerate.

None of this requires running the firmware. None of it requires an exploit. It is structured data, and you read it.

---

## What we learned by dissecting `aopfw-rose.macho`

The dissection in [`MACHO_DISSECTION.md`](MACHO_DISSECTION.md) establishes a single claim with four supporting observations:

**The claim:** `aopfw-rose.macho` is not a simple driver. It is a complete operating system for the AOP2 coprocessor — and it services the most privacy-sensitive sensors on the A18.

**Observation 1 — Kernel-class scaffolding.** The segment map contains a boot stub, its own MMU page tables, four discrete stack contexts (init, IRQ, exception, extended), a 209 KB heap, threading support, and power management. These are the structural pieces of a kernel, not a device driver. A driver is handed memory by a kernel; a kernel builds its own memory.

**Observation 2 — Compartment-aware architecture.** The image contains a `__gxf_data` section (Guarded Execution Framework — the hardware-isolation primitive Apple uses for privileged memory regions), a `__cmevent` section (compartment events), and a separate `__CMA __cma_log_string` region mapped at a high virtual address to isolate a second log domain. The firmware is written to participate in a compartment-based security model, not a flat-memory microcontroller model.

**Observation 3 — Runtime mutation surface.** A section named `_rtk_patchbay` holds 879 bytes of runtime patch table. A patchbay is a sanctioned mechanism for post-load code modification — function pointers or trampoline slots that can be overwritten after the image is loaded. Its existence tells you the firmware was designed with post-load behavioral modification as a normal mode of operation, not an anomaly.

**Observation 4 — Embedded subsystem firmwares.** Parsing the `__cstring` section reveals ten distinct PROGRAM/PROJECT tags — meaning the single `aopfw-rose.macho` file is a bundle containing ten separate firmware components. Among them: `VoiceTrigger64V1` (always-on microphone / "Hey Siri"), `AOPTouchFirmwareiPhone13` (touch digitizer), `DopplerFirmwareiPhone13` (Doppler presence sensing), `AOPALSDriverCT725` (ambient light), `SPUMotion8101` (motion sensing), `iphone13AOPAudio`, plus Bluetooth and UWB subsystems surfaced via runtime log strings. Every one of these is a privacy-sensitive sensor.

---

## Why dissection matters for the A18 flaw

The A18 die flaw, as described in [`FLAW.md`](FLAW.md), does not require looking inside the firmware at all. The wiring alone — documented in the device tree from any `sysdiagnose` — is sufficient to prove that AOP2 has ownership of the sensors, DMA reach into the Exclave, and no external observer.

The Mach-O dissection answers a different question: **what kind of code is Apple actually putting on that coprocessor?**

The answer is: a complete operating system with compartment support and a runtime modification surface, hosting ten distinct sensor-facing firmware components, built 15–22 days before the device capture that exposed it.

The dissection does not prove the flaw. The wiring proves the flaw. The dissection confirms that the thing sitting on the flawed side of the wiring is large, capable, and exactly as structurally privileged as the wiring would allow it to be. If AOP2 had been running a tiny, auditable, single-purpose blob, the structural flaw would still exist but its weaponization surface would be smaller. The dissection shows the weaponization surface is maximal.

In short: the wiring says *the door exists*. The dissection says *the room behind the door is a full operating system with ten live sensors already in it*.

---

## Reproducing the dissection

Anyone with a copy of `aopfw-rose.macho` and Python can reproduce the dissection in minutes. The parsing code is short — under 50 lines of standard `struct.unpack` calls against the Mach-O format, which is fully documented in Apple's `<mach-o/loader.h>` header. No proprietary tools, no reverse engineering frameworks, no disassembler required. The structure is the structure. You read it.

---

**The short version:** a Mach-O is Apple's standard file format for code. Dissecting one means parsing its structure to see what is inside. For the A18 AOP2 firmware, the dissection reveals a complete operating system, a runtime patch surface, and ten embedded sensor firmwares — which is what you would need to see in order to understand what the die flaw actually puts in an adversary's hands.
