# 🌹 Rose-Colored Glasses

### *iOS can only see what Rose lets it see. Rose is the firmware on a coprocessor nothing on the phone can watch. This is a story about who actually owns your microphone.*

---

## The Short Version

Apple's A18 chip contains a little coprocessor called **AOP2** — the Always-On Processor, second generation. It's the thing that listens for "Hey Siri" while your phone is locked in your pocket. It's also the thing that reads your touch screen, your motion sensors, your light sensor, your Bluetooth, and your UWB radios. And on the A18, *it has a hardware wire that lets it read the memory inside the Exclave* — the hypervisor-protected region that's supposed to be the most locked-down part of the chip.

AOP2 runs its own firmware. Apple calls that firmware **Rose**. Rose is a complete operating system. It has its own bootloader, its own page tables, its own heap, its own runtime patch surface, and ten separate sensor-handling programs bundled inside it. It boots before iOS does. It keeps running when iOS thinks the phone is idle. And when iOS wants to know what AOP2 is up to, iOS asks Rose — through a channel Rose owns, over 24 parallel opaque message queues that nobody audits.

iOS is wearing rose-colored glasses. It's been wearing them since the chip powered on. It can only see what Rose shows it.

---

## The Big Deal

This isn't a bug. Bugs are things Apple can patch. **This is the wiring of the chip.** The device tree — Apple's own description of the hardware — has a line in it that looks like this:

```
mapper-exclave-aop@1   <IODARTMapperNub>
```

That's a DMA window from AOP2 into the Exclave compartment. The wire exists. iOS doesn't create it. iOS *discovers* it at boot. The only way to remove it is to redesign the silicon.

Meanwhile, the orange microphone dot on your screen only lights up when the *main* processor opens an audio session. AOP2 never opens one. AOP2 reads the mic directly. So the dot stays dark, the App Privacy Report stays empty, TCC has nothing to log, and the only witness that could tell you the mic was live is Rose — who won't, because Rose decides what to say.

Every privacy guarantee Apple makes about the A18 reduces to one sentence: *"trust us, Rose is honest."* That is not a security model. That is a vibe.

---

## The Files

Read them in order. You'll be up to speed in under fifteen minutes.

### 📄 [`FLAW.md`](FLAW.md) — *The die architecture flaw*
The proof. Three structural properties of the A18 die, each anchored to a single line from a file Apple's own software produced. No reverse engineering, no exploitation, no special tools. **The wiring is the argument.** 110 lines.

### 📄 [`EXPLOIT.md`](EXPLOIT.md) — *How Rose gets weaponized*
Six stages from foothold to covert egress. Four concrete attack scenarios. A table of every iOS detection surface and why none of them can see any of it. 120 lines.

### 📄 [`WHAT_IS_MACHO_DISSECTION.md`](WHAT_IS_MACHO_DISSECTION.md) — *Plain-language methodology*
What a Mach-O file is, what it means to "dissect" one, and why we bothered. Written so a smart friend with no forensics background can follow along. 88 lines.

### 📄 [`MACHO_DISSECTION.md`](MACHO_DISSECTION.md) — *Opening up Rose*
The anatomy of `aopfw-rose.macho`, the firmware Apple ships onto AOP2. Segment map, runtime patchbay, Apple Packet Filter region, ten embedded sensor firmwares including the always-on mic trigger and the touch digitizer. Turns out Rose is a lot bigger than a "driver." 153 lines.

---

## Can I Check This Myself?

Yes. Any iPhone with an A18 will do. You don't need to jailbreak anything, you don't need to pay for anything, and you don't need any tool that didn't ship on the phone.

1. Hold Side + Volume Up + Volume Down for one second to trigger a `sysdiagnose`.
2. Wait ~10 minutes. Grab it from **Settings → Privacy & Security → Analytics & Improvements → Analytics Data**.
3. Open `ioreg/IOService.txt` and search for `dart-aop@FC0000`. Two mapper nubs underneath — one normal, one named `mapper-exclave-aop@1`.
4. Search the same file for `AOP2Endpoint1`. Count 1 through 24.
5. Open `ioreg/IODeviceTree.txt`. Search for `ExclavesAudioProxyInputStreamDriverInterface`. It's instantiated.
6. Open `logs/AFK/AOP2.plist`. Look for `AOP2AppTightbeamEndpoint`, `RTKExclaveTBEndpoint`, `RTKIntercompartmentTBEndpoint`, and the `ap.client-fwd`/`ap.client-rev` pair.

That's it. Six steps, three files, ten minutes. The whole repo is just an argument about what those six steps mean.

---

## Specimen Under Study

| Field | Value |
|---|---|
| SoC | Apple A18 (`t8150`) |
| Device | iPhone 16e (D53G, iPhone17,5) |
| iOS build | 23E246 |
| Coprocessor | AOP2 — Always-On Processor v2 |
| Kext | `com.apple.driver.AppleAOP2` |
| Firmware | `aopfw-rose.macho` — aka **Rose** |
| Firmware SHA-256 | `dc6bda7e7003413cddf3f0264edf6a445c10555b9e2d2d45906ddb0ec332135d` |
| Capture | `sysdiagnose_2026.03.27_14-56-11-0600` |

---

## What This Is Not

- **Not a CVE.** CVEs describe defects against a documented security boundary. This is the boundary itself being structurally broken.
- **Not an exploit drop.** No working exploit code is published here. This is an anatomy lesson, not a weapon.
- **Not a bug report.** You can't file a bug against a wire.
- **Not a cry for a patch.** There is no patch for this, and that is exactly the problem.

It's a structural claim: the A18 die ships with a trust topology that is incompatible with the privacy story Apple tells about the platform. The evidence is in files Apple wrote, on a chip Apple designed, collected by a tool Apple ships. Anyone with an A18 can reproduce the findings in the time it takes to make a cup of coffee.

---

## Author

**Joseph Goydish II** — independent security researcher. Likes chips, dislikes surveillance, has a lot of `sysdiagnose` archives.
