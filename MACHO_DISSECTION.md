# aopfw-rose.macho — Firmware Anatomy

**SHA-256:** `dc6bda7e7003413cddf3f0264edf6a445c10555b9e2d2d45906ddb0ec332135d`
**Size:** 2,457,600 bytes
**Type:** Mach-O 64-bit arm64 preload executable, flags `NOUNDEFS`
**Codename:** Rose
**Loaded by:** `com.apple.driver.AppleAOP2` on Apple A18 silicon (`t8150`)

---

## 1. Header

| Field | Value |
|---|---|
| magic | `0xfeedfacf` (MH_MAGIC_64) |
| cputype | `0x0100000c` (CPU_TYPE_ARM64) |
| cpusubtype | `0x0` |
| filetype | `5` (MH_PRELOAD) |
| ncmds | `11` |
| sizeofcmds | `3792` |
| flags | `0x1` (MH_NOUNDEFS) |

`MH_PRELOAD` is the format for firmware that is loaded at a fixed virtual address by a bootloader rather than dyld. There is no symbol table (`LC_SYMTAB nsyms=0`) and no code signature load command. Entry is via `LC_UNIXTHREAD` (flavor 6, count 68 — ARM_THREAD_STATE64).

## 2. Segment Map

```
LC_SEGMENT_64 __TEXT          vmaddr=0x001000000  vmsize=0x10a000  prot=R-X
    __text                    addr=0x001000000  size=0x0dbba4
    __const                   addr=0x00010dbbb0 size=0x0127d4
    _rtk_mtab                 addr=0x00010ee388 size=0x000780      <-- RTKit method table
    __cstring                 addr=0x00010eeb08 size=0x01b281
    __constructor             addr=0x001109d89 size=0x000000
    __chain_starts            addr=0x001109d89 size=0x000000

LC_SEGMENT_64 __DATA          vmaddr=0x00110a000  vmsize=0xfb000   prot=RW-
    _rtk_boot                 addr=0x00110a000 size=0x003000      <-- early boot stub
    _rtk_page_tables          addr=0x00110d000 size=0x006000      <-- AOP2 MMU page tables
    _spu_stack                addr=0x001113000 size=0x002800      <-- SPU stack
    _rtk_init_stack           addr=0x001116000 size=0x002000
    _rtk_irq_stack            addr=0x001118000 size=0x001000
    _rtk_exc_stack            addr=0x001119000 size=0x001000
    _rtk_ext_stack            addr=0x00111a000 size=0x001800
    _rtk_heap                 addr=0x00111b800 size=0x034750      <-- 209 KB heap
    __const                   addr=0x00114ff50 size=0x012988
    __data                    addr=0x0011628d8 size=0x024ae0
    _rtk_patchbay             addr=0x0011873b8 size=0x00036f      <-- runtime sticker table
    __version                 addr=0x001187728 size=0x000008
    _spu_service              addr=0x001187730 size=0x000870      <-- SPU services
    _spu_endpoint             addr=0x001187fa0 size=0x000090      <-- SPU endpoint
    _rtk_power                addr=0x001188030 size=0x000368
    __cmevent                 addr=0x001188398 size=0x0005a0      <-- compartment events
    __gxf_data                addr=0x001188938 size=0x000010      <-- Guarded eXecution Framework
    _rtk_tunables             addr=0x001188948 size=0x0001e8      <-- runtime tunables
    __mod_init_func           addr=0x001188b30 size=0x000130
    _rtk_data_uuid            addr=0x001188c60 size=0x000000
    _rtk_threads              addr=0x001188c60 size=0x000000
    __zerofill                addr=0x001188c80 size=0x07bd60      <-- 495 KB BSS

LC_SEGMENT_64 __ETEXT         vmaddr=0x001205000  vmsize=0x35000   prot=R-X
    __text                    addr=0x001205000 size=0x02a3b0      <-- separate text region
    __const                   addr=0x00122f3b0 size=0x000726
    __StaticInit              addr=0x00122fad8 size=0x00a12c
    __eh_frame                addr=0x001239c08 size=0x000040

LC_SEGMENT_64 __EDATA         vmaddr=0x001245000  vmsize=0x00000  prot=RW-
LC_SEGMENT_64 __OS_LOG        vmaddr=0x0fd000000  vmsize=0x0d000  prot=RW-
    __string                  addr=0x0fd000000 size=0x00cc84      <-- 51 KB log format strings
LC_SEGMENT_64 __MISC          vmaddr=0x0fe000000  vmsize=0x01000  prot=RW-
    __apf_list                addr=0x0fe000000 size=0x0000a0      <-- Apple Packet Filter rules
LC_SEGMENT_64 __CMA           vmaddr=0x0ff000000  vmsize=0x04000  prot=RW-
    __cma_log_string          addr=0x0ff000000 size=0x0037e2      <-- compartment-managed log
LC_SEGMENT_64 __DATA_CONST    vmaddr=0x001245000  vmsize=0x00000  prot=RW-
```

## 3. What These Segments Mean

This is not a single device driver. It is a **complete RTKit operating system** with its own kernel-grade primitives:

| Section | Implication |
|---|---|
| `_rtk_boot` | AOP2 has its own boot stub. It boots itself from this image at power-on, before iOS exists. |
| `_rtk_page_tables` | AOP2 has its own MMU and page tables. It enforces its own memory protection. |
| `_rtk_init_stack`, `_rtk_irq_stack`, `_rtk_exc_stack`, `_rtk_ext_stack` | Four discrete stack regions for init, IRQ, exception, and extended contexts — kernel-class architecture. |
| `_rtk_heap` (209 KB) + `_rtk_threads` | Dynamic allocation and threading inside AOP2. |
| `_spu_stack`, `_spu_service`, `_spu_endpoint` | Embedded **SPU** (Secure Processing Unit) integration — AOP2 talks directly to the SPU silicon block. |
| `_rtk_patchbay` (0x36F bytes) | **Runtime sticker table.** A 62-entry `GKTS` TLV table holding per-device and per-boot configuration — identity (ECID, nonce, KASLR slide, PRNG seed), runtime layout, behavior flags, IO base addresses, and a pointer to `_rtk_tunables`. The loader writes into this region at bring-up, before AOP2 begins executing. Full enumeration in [`PATCHBAY.md`](PATCHBAY.md). |
| `_rtk_tunables` | Runtime-tunable parameters — behavior can be changed without reflashing. |
| `__cmevent` (compartment events, 0x5A0 bytes) | The firmware emits and consumes events scoped to security compartments. |
| `__gxf_data` | **Guarded Execution Framework** data region. GXF is the same hardware-isolation primitive used in PPL and Exclave enforcement. AOP2 firmware participates in GXF. |
| `__OS_LOG __string` (51 KB) | AOP2 emits OS log entries via the firehose — string IDs only, the format strings live here. The AP-side `tracev3` parser resolves these. |
| `__MISC __apf_list` | **Apple Packet Filter** rule list. AOP2 has a BPF-class packet filter VM. |
| `__CMA __cma_log_string` | A second log string region in a separate compartment ("CMA"). Two log domains, two compartments. |
| `__ETEXT` (extended text, 0x2A3B0 bytes) | A second executable segment, separately mapped. This is consistent with a compartment boundary between primary firmware and an extended-privilege payload. |

## 4. Embedded Subsystem Firmwares

`aopfw-rose.macho` is a multi-program preload bundle. Embedded program/project tags found in `__cstring`:

| Program | Project | What it is |
|---|---|---|
| `SPUMotion8101` | `CoreLocation-3072.0.46.0.2` | Motion sensing on the SPU |
| `VoiceTrigger64V1` | `AppleAOPVoiceTriggerFirmware-540.5` | "Hey Siri" / voice trigger — always-on mic capture |
| `iphone13AOPAudio` | `AppleAOPAudioFirmware-540.66` | AOP audio firmware |
| `RTKAudioDriversT8101_RTK` | `RTKitAudioDrivers-540.19` | RTKit audio drivers |
| `AudioProviderFramework` | `AudioProviderFramework-*` | AOP audio provider |
| `RTKitAudioFramework_64` | `RTKitAudioFramework-540.4` | RTKit audio framework |
| `DopplerFirmwareiPhone13` | `DopplerFirmware-104.0.0` | Doppler radar / motion (audio-Doppler presence) |
| `AOPALSDriverCT725` | `AppleEmbeddedLightSensor-2079.100.112` | Ambient light sensor |
| `AOPTouchFirmwareiPhone13` | `AOPTouchSupport-313` | **Touch digitizer firmware on AOP2** |

Build banners:
```
AppleAOPVoiceTriggerController v540.5 built: Mar 12 2026 20:14:29
AppleAOPAudioFirmware-540.66~2734  built: Mar 12 2026 20:48:04
RTKitAudioFramework-540.4~2150     built: Mar  5 2026 23:10:35
```

Build dates are recent relative to the sample capture — current production firmware.

## 5. The Sensor Reach

The presence of **VoiceTrigger**, **AOPAudio**, **AOPTouch**, **AOPALS**, **Doppler**, **SPUMotion**, **CoreLocation**, plus the BT/UWB/FiRa logic surfaced via `aoprose` log strings, means the AOP2 firmware actively services:

- Microphone (always-on, voice trigger)
- Touch input (digitizer)
- Ambient light
- Motion / accelerometer
- Doppler (audio-Doppler presence detection)
- Location
- Bluetooth
- Ultra-wideband (FiRa precision ranging)
- Nearby Interaction passive access intents (`NISystemPassiveAccessIntentGeofenceEntry`)

All of this runs **on AOP2**, with AOP2's own page tables, AOP2's own heap, AOP2's own GXF compartment, AOP2's own DART (`dart-aop@FC0000`), and a DMA window into the Exclave (`mapper-exclave-aop@1`). The AP receives only what AOP2 chooses to forward through one of the 24 `AFKAOP2EndpointN` channels.

## 6. The Patchbay

`_rtk_patchbay` at `0x0011873b8` is 0x36F bytes (879 bytes) of patch table. A patchbay in RTKit is a set of slots that allow runtime code substitution — function pointers or trampoline addresses that the firmware reads after boot to redirect calls. The patchbay is *inside the firmware image*, meaning patches are signed alongside the firmware, but the existence of a patchbay region means there is a sanctioned mechanism to alter behavior post-load without rebuilding the image. For an investigator: the patchbay should be enumerated and each entry resolved against `__text` to determine which functions are interposable.

## 7. The Compartment Layout

Three high-virtual-address segments are mapped at `0xfd000000`, `0xfe000000`, and `0xff000000`:

- `__OS_LOG` at `0xfd000000`
- `__MISC __apf_list` at `0xfe000000`
- `__CMA __cma_log_string` at `0xff000000`

These are not contiguous with the main `__TEXT`/`__DATA` blob (which lives at `0x01000000`). The 4-bit-aligned high-address layout is consistent with **separate compartment views** — different VM windows with different access controls. The presence of `__cmevent`, `__gxf_data`, and `__cma_log_string` together describes a firmware that participates in a compartment-aware execution model, not a flat-address microcontroller blob.

---

**Status:** firmware structure documented. `_rtk_patchbay` enumerated in full — see [`PATCHBAY.md`](PATCHBAY.md). Remaining work: resolve `__apf_list` rules, dump `__OS_LOG __string` against the AP-side `tracev3` uuidtext catalog to confirm which AOP2 messages reach the AP firehose.
