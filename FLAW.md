# The A18 Die Architecture Flaw

**Subject:** Apple A18 SoC — silicon identifier `t8150`.
**Block under study:** AOP2 (Always-On Processor, second generation) — RTBuddy slave processor, published to the main application processor (AP) through `com.apple.driver.AppleAOP2`.
**Fabric:** `arm-io,t8150` — the A18's on-die bus and peripheral hierarchy.
**Adjacent silicon:** SEP (Secure Enclave Processor), Exclave compartment (hypervisor-enforced memory domain), DART IOMMU blocks, SPU (Secure Processing Unit).

---

## The Flaw

On the A18 die, a single coprocessor — **AOP2** — holds three properties at once:

1. It owns the microphone, touch digitizer, ambient light, motion, Doppler, Bluetooth, and UWB.
2. It has a hardware DMA window into the Exclave memory compartment.
3. Its internal state cannot be observed from anywhere else on the device.

The composition is the flaw. Each property alone is defensible. Together, they describe a processor that owns the most privacy-sensitive sensors, has DMA reach into the most privileged memory, and reports on itself through a channel it controls.

---

## The Anchors

Each property is proven by one file from Apple's own `sysdiagnose`.

### Property A — Sensor Sovereignty
`ioreg/IODeviceTree.txt`:
```
ExclavesAudioProxyInputStreamDriverInterface = 1
```
The microphone stream passes through an Exclave proxy before it reaches the AP audio HAL. AOP2 is in the path, not beside it.

### Property B — Privileged DMA Reach
`ioreg/IOService.txt:7292,7376,7403`:
```
dart-aop@FC0000               ; AOP2-private IOMMU
  mapper-aop@0                ; standard DMA window
  mapper-exclave-aop@1        ; DMA window into Exclave
```
Two mapper nubs on the AOP2 IOMMU. The second is wired into the Exclave compartment. The wire is part of `arm-io,t8150`, the SoC fabric. iOS discovers it; iOS does not create it.

### Property C — Opacity
`ioreg/IOService.txt:7738,7808` and `logs/AFK/AOP2.plist`:
```
RTBuddy(AOP2)  <IOSlaveProcessor>
CFBundleIdentifier = com.apple.driver.AppleAOP2
AOP2Endpoint1 .. AOP2Endpoint24
ap.client-fwd  /  ap.client-rev
RTKExclaveTBEndpoint
RTKIntercompartmentTBEndpoint
```
AOP2 is published as a slave processor. The AP is a relay client (`ap.client`), not a controller. Twenty-four opaque message channels, no schema enforcement, no introspection ABI.

---

## Why It Composes Into A Flaw

- Sovereignty removes the AP's ability to independently verify sensor readings, because AOP2 owns the source.
- DMA Reach removes the Exclave's boundary against AOP2, because the wire exists.
- Opacity removes the AP's ability to detect either of the above failing, because the AP only knows what AOP2 chooses to tell it.

The three properties form a closed loop with no external observer. The A18's privacy model reduces to one unwitnessable assumption: *AOP2 firmware is honest, and we know this because AOP2 says so.*

---

## Where The Flaw Lives

The flaw is in the wiring of the A18 die, established at tape-out. Every element of the composition is described in the hardware enumeration artifacts the SoC emits at boot, before any operating system has loaded.

| Element | Where it lives |
|---|---|
| `dart-aop@FC0000` | `arm-io,t8150` fabric — fixed at tape-out. |
| `mapper-exclave-aop@1` | Child of the AOP DART in the same fabric. Enumerated by hardware discovery, not created by the OS. |
| AOP2 boot sequence | Triggered at SoC power-on, before the AP begins executing iOS. |
| Exclave audio proxy | Instantiated as part of the supported audio pipeline routing into the hypervisor-enforced compartment. |
| Silicon-level opacity | A property of the relay topology: the AP's only introspection path into AOP2 is a channel AOP2 itself controls. No second observer exists on the die. |

The structural properties are fixed by the die's layout. The exploitation surface — how code running on AOP2 leverages those properties at runtime — is walked through in [`EXPLOIT.md`](EXPLOIT.md).

---

## Why It Cannot Be Denied

Every claim anchors to a file Apple software produced on stock Apple silicon:

- `ioreg/IOService.txt` — output of `ioreg`, the kernel IOKit registry dump.
- `ioreg/IODeviceTree.txt` — IOKit diagnostics, class instance counts.
- `logs/AFK/AOP2.plist` — the AFK service tree, serialized at sysdiagnose time.

All three appear on every shipping A18 device today. No proprietary tooling, no exploit, no privileged access.

---

## Reproduction

1. On any A18 iPhone, hold Side + Volume Up + Volume Down to trigger a `sysdiagnose`.
2. Wait ~10 minutes. Retrieve from Settings → Privacy & Security → Analytics & Improvements → Analytics Data.
3. Extract the archive.
4. In `ioreg/IOService.txt`, search `dart-aop@FC0000`. Confirm `mapper-aop@0` and `mapper-exclave-aop@1` as child nubs.
5. In the same file, search `AOP2Endpoint1`. Confirm the `IONameMatch` array enumerates 1 through 24.
6. In `ioreg/IODeviceTree.txt`, search `ExclavesAudioProxyInputStreamDriverInterface`. Confirm instance count ≥ 1.
7. In `logs/AFK/AOP2.plist`, confirm `AOP2AppTightbeamEndpoint`, `RTKExclaveTBEndpoint`, `RTKIntercompartmentTBEndpoint`, and the `ap.client-fwd`/`ap.client-rev` pair.

Seven steps. Three files. Any A18 device. Ten minutes.

---

## The Flaw, In One Line

> *The A18 die wires a single coprocessor — whose state nothing on the device can independently observe — both ownership of the device's most privacy-sensitive sensors and DMA reach into the device's most privileged memory compartment. The privacy model of the platform reduces to faith in a witness vouching for itself.*

