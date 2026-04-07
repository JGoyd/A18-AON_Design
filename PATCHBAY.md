# The Rose Patchbay, Enumerated

`_rtk_patchbay` in `aopfw-rose.macho` at file offset `0x001883b8`, size 879 bytes (`0x36f`). Walked byte by byte. Parses as a fully-enumerated table with zero leftover bytes.

---

## What It Is

The section is a **sticker table**, not a function-trampoline table. Stickers are Apple's format for per-device, per-boot runtime configuration written into a firmware image by whichever component loads the firmware. The section name `_rtk_patchbay` describes what it does from Rose's point of view — *it patches runtime state post-load* — but what it patches is **data**, not code.

The table opens with a `GKTS` FourCC header (`"STKG"` in little-endian), two header fields, and then 62 TLV entries. Each entry is:

```
tag        4 bytes    FourCC (stored reversed in memory)
size       4 bytes    payload size in bytes
payload    size bytes value
```

62 entries, 879 bytes, exact fit. No alignment slack, no trailing padding. The format is correct.

Full enumeration: [`evidence_patchbay.txt`](evidence_patchbay.txt).

---

## What Is In It

The 62 entries fall into six categories.

### Per-device identity (filled at boot)

| FourCC | Size | Populated by |
|---|---|---|
| `ECID` | 8 | Exclusive Chip ID — silicon serial, written per-device |
| `NONC` | 8 | Boot nonce, written per-boot |
| `SLID` | 8 | KASLR slide, written per-boot |
| `PRNG` | 32 | PRNG seed, written per-boot |

All four ship as zero in the image. The loader fills them during bring-up.

### Runtime layout (fixed at build)

| FourCC | Size | Value |
|---|---|---|
| `RTSZ` | 4 | `0x00205000` — runtime size |
| `GptS` | 8 | `0x00004000` — GPT size |
| `TUNS` | 8 | `0x01188948` — pointer to `_rtk_tunables` |
| `TUNZ` | 4 | `0x000001e8` (488) — size of `_rtk_tunables` |
| `McRA` | 4 | `0x00004000` — 16 KB region |

`TUNS` and `TUNZ` point at and size the `_rtk_tunables` section exactly — `_rtk_tunables` lives at `0x01188948` with size `0x1e8`. The sticker table is a **front door to the tunables table**.

### Runtime behavior flags (mostly zero, loader-writable)

| FourCC | Size | Value | Purpose |
|---|---|---|---|
| `RTLL` | 1 | `1` | Runtime log level |
| `DEVf` | 1 | `0` | Dev-mode flag |
| `IGNT` | 1 | `0` | Ignition / boot gate |
| `adbg` | 4 | `0` | Debug enable |
| `pFLG` | 8 | `0x2` | Platform flags |
| `TTTM` | 4 | `1` | (unresolved) |
| `napE` | 4 | `1` | Nap enable |
| `nCal` | 1 | `1` | Cal state |

### IO base / memory windows (zero-filled placeholders)

`IOBA`, `IOSZ`, `WrAd`, `CpAd`, `BPTP`, `SPTP`, `BVTP` — eight-byte slots for memory-mapped IO bases, copy/write targets, and page-table bases. All zero in the image; written by the loader.

### Peripheral bring-up config

`ri2c` / `si2c` (I²C read/send), `aclH`, `bbOn`, `brdI`, `MCFG` (memory config), `DSCL`, `DSVS`, `DSRL` (DSC-prefixed), `PACE`, `CBES`, `ZSTR` (`0x00205000`). Hardware-facing peripheral parameters, populated per-SoC-revision.

### Clock / power / boot path

`EC0p`, `ED0p` (power rails), `T1Ps`, `v8Fq` (frequency), `bFST` (boot fast), `k2LP` / `k2PI`, `GptB`, `PplB`, `DSZS` / `DSZL`, `S2xG`, `GFCM`, `SSSC`, `SRSA`, `ECAP`, `RTTO`, `DICE`, `PTPB` / `PTPS`, `SNeo`, `LZSD` / `SZSD`. Power, clock, boot, and cryptographic parameter slots.

---

## What The Enumeration Tells Us About Rose

### 1. The runtime is built to be reconfigured at load time, not rebuilt

Forty-five of the sixty-two slots ship as zero. The loader is expected to write into them. Rose's behavior — log level, debug state, dev mode, IO bases, KASLR slide, runtime sizes, tunable pointers — is not fixed at build time. It is finalized at boot by whichever component loads the image.

That component is not the AP kernel. Rose boots from its own image at SoC power-on, before the AP is executing iOS. The writes into the sticker table happen before any AP-side observer exists.

### 2. The loader has in-image write access

The `__DATA` segment containing `_rtk_patchbay` has memory protections `RW-`. The writes into the sticker table happen into the live `__DATA` region after Rose is placed in memory but before it executes. The loader reaches into Rose's own data space and modifies identified slots. This is a supported, designed, and required part of Rose's bring-up.

### 3. Behavior is a dial the loader turns

The `RTLL` (log level), `adbg`, `DEVf`, `IGNT`, `napE`, and `pFLG` slots are runtime behavior knobs. A loader that writes `RTLL=0` produces a Rose that emits fewer log messages. A loader that writes `DEVf=1` produces a dev-mode Rose. A loader that writes `adbg=1` produces a debug-enabled Rose. These are not hypothetical modifications — they are the exact fields the sticker format was defined to hold. The fields exist for this reason.

### 4. The sticker table is the official post-signing modification surface

Rose as-shipped carries no code signature load command (`LC_CODE_SIGNATURE` is absent from the Mach-O header). Integrity is enforced by whatever outer wrapper loads the image, not by the Mach-O itself. Even so, the sticker table would sit below any signature boundary because **the sticker format requires post-sign writes to function**. Per-boot nonces, per-device ECIDs, and per-boot KASLR slides cannot be part of a statically-signed blob — they are by definition values that differ on every boot and every device. The sticker table is therefore an architecturally exempt region.

### 5. `TUNS`/`TUNZ` resolve the second modification surface

The sticker table holds a pointer to `_rtk_tunables` (488 bytes, at `0x01188948`). Whatever is in `_rtk_tunables` is reached via this pointer. A second enumeration is now possible against those 488 bytes.

---

## What The Enumeration Does Not Tell Us

- It does not tell us whether any specific loader has written non-default values into a given slot on a given device.
- It does not tell us which loader component is responsible for the writes (iBoot? a trusted bring-up chain component? SEP? something else).
- It does not identify function-level interposition hooks. Those are not in this section. If Rose has a function-trampoline mechanism, it lives elsewhere — candidates include `__mod_init_func`, `_rtk_mtab`, or per-image late binding during RTKit init.
- It does not name what `TTTM`, `SSSC`, `SNeo`, `LZSD`, and several other acronyms mean. Those would require cross-reference to public iBoot / RTKit research or to additional firmware images.

---

## The Cut-And-Dry Result

> **Rose's `_rtk_patchbay` section is a 62-entry sticker table at file offset `0x001883b8` holding runtime configuration, not function interposition slots. The table's format (`GKTS` TLV), its size (879 bytes, exact fit), its field inventory (identity, layout, behavior flags, IO bases, peripheral config), and its relationship to `_rtk_tunables` (direct pointer) are all resolved. Forty-five of sixty-two slots are zero in the image and are written by the loader at boot. The behavior of Rose — log verbosity, debug state, dev mode, boot gating, runtime memory layout — is not fixed at build time. It is a dial the loader turns before the application processor is running iOS.**

The modification surface established by this section is real, but its shape is different from the code-hook surface implied earlier in [`EXPLOIT.md`](EXPLOIT.md). The code-hook surface, if it exists, is elsewhere in the image. This section is the **runtime parameter surface** — narrower, but more rigorously enumerated than anything else in the repo.

---

## Evidence

- [`evidence_patchbay.txt`](evidence_patchbay.txt) — complete 62-entry dump with tag, reversed FourCC, size, raw hex payload, and typed interpretation.
