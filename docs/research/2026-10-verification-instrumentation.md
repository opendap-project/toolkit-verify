# Research: verification instrumentation for every OpenDAP technical promise

- **Date:** 2026-10-22 (brief dated 2026-10-21)
- **Status:** Research. Every proposal here is a **recommendation, not a decision**. Nothing in this document is a confirmed requirement or ADR.
- **Owner directive that drives this report:** "If we promise DSD/DoP and bit-perfect, it must be verified by software at least, not only theoretical, until we can measure it."
- **Scope:** every **Confirmed** or **Draft** requirement under `public/*/docs/requirements/` and `public/*/*/docs/requirements/`, plus the accepted ADRs and specs (`t1/docs/specs/faceplate-interface.md`, `toolkit/hardware/docs/specs/component-export.md`). Withdrawn and Moved requirements are ignored.
- **Constraints taken as given:** no CI, everything runs locally; UNIX/POSIX only, tested on macOS (TK-010, TK-011); agentic-first CLI (TK-009, conventions in [`2026-10-agentic-cli-design.md`](https://github.com/opendap-project/toolkit-cli/blob/main/docs/research/2026-10-agentic-cli-design.md) §8); the virtual T1 in `blackbird-os/build` (QEMU 11.1.2 `virt` + HVF, 480 × 800 virtio-gpu, virtio-multitouch, virtio-keyboard buttons, virtio-sound, QMP via `build/dap-qmp.py`, SSH on 2222).
- **Evidence convention:** a fact with a link was read from that primary source in this session (2026-10-21/22). **UNVERIFIED** marks anything not confirmed against a primary source. Tool versions already checked by the sibling reports ([hardware gap analysis](https://github.com/opendap-project/toolkit-hardware/blob/main/docs/research/2026-10-hardware-toolchain-gap-analysis.md), [co-design workflow](https://github.com/opendap-project/toolkit-hardware/blob/main/docs/research/2026-10-co-design-workflow.md), [software/firmware toolchain](https://github.com/opendap-project/toolkit-host/blob/main/docs/research/2026-10-software-firmware-toolchain.md), compliance, [`blackbird-os` image composition](https://github.com/opendap-project/blackbird-os/blob/main/docs/research/2026-10-image-composition.md), [`blackbird-os` OS lifecycle](https://github.com/opendap-project/blackbird-os/blob/main/docs/research/2026-10-os-lifecycle.md)) are reused and cited to those reports rather than re-checked.

---

## 0. Key findings (read this first)

1. **The virtual T1's audio device cannot be the bit-perfect oracle.** QEMU's `virtio-snd` advertises only S8/U8/S16/U16/S32/U32/FLOAT and no 24-bit or DSD formats; its source has no mention of DSD ([`hw/audio/virtio-snd.c`](https://raw.githubusercontent.com/qemu/qemu/master/hw/audio/virtio-snd.c)). Its host side also ends in CoreAudio, which may convert. So "bit-perfect in the VM" must be checked **at the guest's ALSA boundary**, not at the host speaker.
2. **`snd-aloop` is the right software sink, and it supports DSD.** The mainline loopback driver declares S16/S24/S24_3/S32 (LE and BE), FLOAT, **DSD_U8, DSD_U16_LE/BE and DSD_U32_LE/BE**, continuous rates **8 kHz–768 kHz** and 1–32 channels ([`sound/drivers/aloop.c`](https://raw.githubusercontent.com/torvalds/linux/master/sound/drivers/aloop.c)). The playback engine opens `hw:Loopback,0`; a capture harness reads `hw:Loopback,1` and writes exactly what the engine sent. This makes **PCM, native DSD and DoP bit-perfect verifiable today, in the virtual T1, with no hardware.**
3. **The loopback proves the engine, not the T1 audio hardware.** `snd-aloop` accepts any rate, so it cannot prove that the real I2S/clock tree plays 44.1k-family rates exactly (BRD-023). That part needs a constrained fake device (§3.4) now and a clock measurement on hardware later. Reports must say which of the two was verified.
4. **Negative controls are mandatory.** Each audio check must also run a path that is *known* to break bit-perfection (PipeWire's graph is always float 32, per [pipewire-props(7)](https://docs.pipewire.org/page_man_pipewire-props_7.html)) and must report FAIL there. A 32-bit PCM vector with non-zero low 8 bits, and a DoP stream, are the cheapest probes. A harness that never fails proves nothing.
5. **DoP and DSF have exact, testable bit layouts from primary sources.** DoP v1.1: 8-bit marker in the MSBs alternating **0x05 / 0xFA** per sample, same marker on every channel of a frame, 16 DSD bits below it with the oldest bit first, PCM rate 176.4 kHz for DSD64 and 352.8 kHz for DSD128 ([DoP open standard v1.1](https://dsd-guide.com/sites/default/files/white-papers/DoP_openStandard_1v1.pdf)). DSF stores 1-bit data **LSB first**, in **4096-byte blocks per channel**, and zero-fills the unused part of the last block ([Sony DSF spec](https://dsd-guide.com/sites/default/files/white-papers/DSFFileFormatSpec_E.pdf)). ALSA's DSD silence byte is **0x69** ([`sound/core/pcm_misc.c`](https://raw.githubusercontent.com/torvalds/linux/master/sound/core/pcm_misc.c)). A validator must handle all three: bit reversal, block de-interleaving and the zero padding, which is *not* DSD silence.
6. **Faceplate HID commands can be verified end-to-end in the virtual T1 through UHID.** `/dev/uhid` creates kernel HID devices from userspace; `struct uhid_create2_req` carries `bus`, `vendor`, `product`, `version` and the report descriptor ([`include/uapi/linux/uhid.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/linux/uhid.h), [UHID docs](https://www.kernel.org/doc/html/latest/hid/uhid.html)). Feeding the core-set descriptor through it checks the spec's "usage → Linux event" table against the real kernel of the image. It does **not** exercise the `i2c-hid` transport, which needs a host-side protocol test now and a logic analyser later.
7. **No usable RP2350 emulator was found.** Renode 1.17.0 release notes do not mention RP2040/RP2350 ([Renode releases](https://github.com/renode/renode/releases)); `rp2040js` is RP2040-only ([GitHub](https://github.com/wokwi/rp2040js)). **Recommendation:** a real Raspberry Pi Pico 2 on the dev host is the stand-in for BRD-015/OS-008 now, driven by `picotool` (RP2350 support, `load`, `verify`, partition tables, per its [README](https://raw.githubusercontent.com/raspberrypi/picotool/master/README.md)) and by a second Pico running `debugprobe` as a CMSIS-DAP SWD probe ([GitHub](https://github.com/raspberrypi/debugprobe)).
8. **Battery emulation in the VM has a kernel gap.** `test_power` has runtime-writable parameters for capacity, voltage, current, status and charge counter ([`test_power.c`](https://raw.githubusercontent.com/torvalds/linux/master/drivers/power/supply/test_power.c)), but the Debian kernel does not build it (recorded in `blackbird-os` requirements context). Userspace cannot create `power_supply` devices, so **battery logic must be testable behind an injectable source** (a sysfs-root fixture or a bus-level fake), and the real fuel gauge is verified only on hardware.
9. **A/B rollback can be tested in QEMU with the same U-Boot logic as the device.** U-Boot's `qemu_arm64_defconfig` boots the `virt` machine with `-bios u-boot.bin` and can keep its environment in a pflash image ([U-Boot QEMU docs](https://docs.u-boot.org/en/latest/board/emulation/qemu-arm.html)); the lifecycle research already proposes this "fidelity path" with `bootmeth_rauc`. The current virtual T1 boots EDK2, so this path is still to be built.
10. **Several promises can be verified *statically* now and nobody does it yet:** mainline-driver availability for every chosen part (BRD-001, OS-002, FPT-018) by checking `modules.alias`/`modules.builtin` of the pinned Debian kernel package; the privacy-LED wiring rule (BRD-027) on the netlist; the footprint (T1-013) and coordinate convention (T1-021) on exported STEP files; kernel ≥ 7 (OS-001) from the image manifest.
11. **Design-for-test decisions are due now, before the board is drawn** (§9): test pads on MCLK/BCLK/LRCK/SDATA, sense resistors or shunt points on the battery and main rails, exposed SWD for the RP2350, the faceplate I2C/INT lines, and the HAT lanes' launch geometry suited to probing. Without them, several promises can never be measured on the T1 itself.
12. **The lab's critical instruments are few and affordable**, except high-speed SI: QA403 audio analyser **US$599** ([QuantAsylum](https://quantasylum.com/collections/frontpage)), Joulescope JS220 **US$999** ([Joulescope](https://www.joulescope.com/products/js220-joulescope-precision-energy-analyzer)), a Nordic PPK2 (0.2 µA–1 A, 100 ksps, 0.8–5 V source mode, [Nordic](https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2)), a sigrok-supported logic analyser, a scope and a programmable supply. The ≥ 20 GHz VNA/TDR for PCIe/USB3 over pogo pins should be **rented** (gap analysis finding 5). Caveat: the QA40x application is Windows-native (Linux via Mono, untested; macOS not mentioned) ([QA403 page](https://www.quantasylum.com/products/qa403-audio-analyzer)); it exposes an HTTP REST API, so OpenDAP's macOS scripts should talk to it over the network from a small Windows/Linux host (**UNVERIFIED** whether an open driver exists that runs on macOS).

---

## 1. Method

1. Read every requirement README listed in the scope (board, t1, case, hat-standard, blackbird-os, system, toolkit, kicad-lib, ui-components, blender-lib, freecad-lib, faceplates/{touch,classic,cassette}, hats/{camera,nas,oscilloscope,powerbank,radio,recorder,vna}), every ADR (`board` 0001–0004, `t1` 0001–0002, `case` 0001–0003, `hat-standard` 0001, `blackbird-os` 0001–0005, `system` 0001) and the two specs.
2. Kept only Confirmed and Draft rows. Grouped them into **24 verifiable promises** (§2).
3. For each promise: defined pass criteria, a software verification now, a hardware measurement later, and the tools (existing vs to build).
4. Checked the load-bearing facts (driver format tables, DoP/DSF layouts, UHID, QMP, test_power, U-Boot QEMU, RAUC, instruments) against primary sources.
5. Did **not** boot the virtual T1 in this session (other agents share it). Every "works in the virtual T1" claim below that is not backed by a source link is therefore **UNVERIFIED until the first run**; §8 lists them as the first checks to execute.

### 1.1 Verification levels used in this report (recommendation)

| Level | Name | Environment | What it can prove |
|---|---|---|---|
| **L0** | Static | Files only: specs, YAML, KiCad netlists, STEP, DT, image manifest, kernel package | Consistency, presence, rules (e.g. a driver exists, a net is wired as required) |
| **L1** | Model / simulation | ngspice, openEMS/Palace + scikit-rf, FreeCAD FEM/CalculiX, Python models | Physics estimates against a budget; ranks design variants |
| **L2** | Virtual T1 | QEMU `virt` + Blackbird OS image, QMP, SSH, `snd-aloop`, UHID, throttled block devices | Software behaviour exactly as shipped, at the kernel/userspace boundary |
| **L3** | Dev-kit hardware | An RK3566 SBC with mainline support, eval boards (DAC, fuel gauge, Wi-Fi module), a Pico 2 | Real drivers, real clocks, real power on the same SoC |
| **L4** | T1 prototype | EVT/DVT boards in the case, with faceplate and HATs | The product promise itself |
| **L5** | Accredited lab | EMC, safety, IP, hearing safety | Certification evidence |

**"Verified by software"** (maintainers' directive) is defined here as: **an L0–L2 check passes, its negative control fails as expected, and the result is recorded against the requirement ID.** The report always states the highest level reached, so a promise verified at L2 is never shown as "measured".

---

## 2. The promises

| # | Promise | Requirement IDs (Confirmed/Draft only) |
|---|---|---|
| P1 | Bit-perfect PCM, every rate, no resampling, no format conversion, no volume/dither at unity | OS-020, BRD-023 (Draft), T1-008, T1-011 |
| P2 | Bit-perfect DSD: native DSD and DoP | OS-020, BRD-022 (Draft) |
| P3 | Two clock families; DAC as clock master; per-track rate switching | BRD-023 (Draft) |
| P4 | Analog audio quality on 3.5 mm SE and 4.4 mm balanced | T1-008, BRD-011, BRD-022 (Draft), BRD-024 (Draft), ADR board-0001 |
| P5 | Speaker driven independently of the DAC | BRD-012, T1-016 |
| P6 | Hearing safety: rolling 7-day sound dose, 85 dB(A), per-headphone profiles | SYS-012 (Draft) |
| P7 | Battery life and power management | T1-009, T1-012, BRD-008, BRD-009, BRD-020 (Draft) |
| P8 | Protection and monitoring | BRD-007, BRD-010, BRD-026 (Draft), HCS-015 (Draft) |
| P9 | Mainline drivers, no custom kernel, kernel ≥ 7, prebuilt kernel updates | BRD-001, OS-001, OS-002, OS-006, OS-007 (Draft), FPT-008 (Draft), FPT-018, BRD-025 (Draft) |
| P10 | Image: built from code, trimmed, signed, versioned, simple flashing | OS-004, OS-005, OS-010, OS-011, OS-012, OS-021, ADR blackbird-os 0004/0005 |
| P11 | Updates, OTA, A/B rollback, recovery | OS-013, OS-014 (Draft), OS-017 (Draft), BRD-016, BRD-017, BRD-018, BRD-035, BRD-039 |
| P12 | Boot without RP firmware; boot time (implicit in product quality, no numeric requirement) | BRD-019 |
| P13 | Virtual T1 fidelity and remote control | OS-009, OS-016, OS-018, OS-019 |
| P14 | Peripheral hosting: Blackbird OS flashes, operates and debugs every chip | OS-008, OS-015, BRD-004, BRD-005, BRD-014, BRD-015, HCS-013, ADR board-0004 |
| P15 | Faceplate controls as standardized HID commands; identification | T1-018, T1-020, T1-023, T1-024, FPK-009, FPC-009, ADR t1-0002 |
| P16 | Display, touch, status LED, ALS, microphones, IMU, buttons | FPT-002, FPT-003, FPT-004, FPT-010 (Draft), FPT-013, FPT-014, FPT-015, FPT-016, FPT-017, FPK-004…008, FPC-004…008, BRD-027 (Draft), BRD-031, BRD-032, BRD-036, T1-014, T1-015, T1-022 |
| P17 | HAT interface: direct high-speed lanes, maximum speed, one interconnect type | HCS-004, HCS-005, HCS-006, HCS-014 (Draft), BRD-006, BRD-013, BRD-033, ADR hat-standard-0001, ADR board-0002 |
| P18 | HAT identity: EEPROM, signed capability manifest, no central registry | HCS-007, HCS-008, HCS-009, HCS-010, HCS-011, HCS-017 (Draft) |
| P19 | Connectivity and storage | BRD-029, BRD-030, BRD-034, BRD-038, OS-022, BRD-037 (Draft) |
| P20 | Mechanics: footprint, coordinates, button layout and click, fixing, materials, IP65, drop | T1-013, T1-017 (Draft), T1-021 (Draft), T1-022, CASE-006, CASE-020, CASE-021, CASE-024, CASE-025, HCS-012, HCS-019, HCS-020 |
| P21 | Co-design: product assembled and simulated from component exports; design rules with test boards | T1-007, KLIB-002 (Draft), component-export spec |
| P22 | System software architecture (capabilities, bus, activities, packages via the package endpoint) | SYS-001…SYS-011 (Draft), DS-001 |
| P23 | Toolkit CLI | TK-001…TK-011, T1-025 |
| P24 | HAT products | CAM-005, NAS-005, NAS-006, OSC-005, PWR-005, RAD-005, REC-005, VNA-005, plus *-001…004 |

Non-technical requirements (open source/licensing T1-001, BLIB-002; editions T1-004…006; sourcing T1-010, BRD-002; libraries KLIB-001, FLIB-001, BLIB-001) are listed in the traceability table with the static check that fits them (for example `reuse lint`), without a deep dive.

---

## 3. P1–P3: bit-perfect PCM, DSD (native and DoP) and rate switching

This is the deepest section because it is the maintainers' explicit directive.

### 3.1 What "verified" means (pass criteria, recommendation)

For one **test vector** played by the first-party engine on one **output path**:

1. **Payload identity.** After alignment, the captured stream contains the reference stream as **one contiguous, byte-identical run**, from the first to the last sample of the track. Everything outside that run is digital silence (all-zero PCM, or the DSD silence byte 0x69, or DoP frames whose payload is 0x69 with valid markers). No sample inside the run may differ: tolerance is **zero bits**.
2. **Format and rate identity.** While the track plays, the PCM substream's `hw_params` (`/proc/asound/card*/pcm*p/sub*/hw_params`, documented as "the hardware parameters set for this sub-stream" in the [ALSA proc docs](https://www.kernel.org/doc/html/latest/sound/designs/procfile.html)) shows:
   - `format` equal to the expected container (e.g. `S24_3LE`, `S32_LE`, `DSD_U32_BE`);
   - `rate` equal to the track's native rate (PCM), the DoP carrier rate (DSD rate ÷ 16) or the native DSD frame rate (DSD rate ÷ bits per container: DSD64 = 2,822,400 bit/s → 352,800 Hz for `DSD_U8`, 176,400 Hz for `DSD_U16_*`, 88,200 Hz for `DSD_U32_*`);
   - `channels` equal to the track's channel count.
3. **Exclusive path.** The substream's `status` shows the engine's PID as owner (**UNVERIFIED** that `owner_pid` is printed on 7.2; the `status` file is documented as "current status … elapsed time, H/W position, etc."), and no PipeWire node is linked to that card during the run (`pw-dump` shows the device reserved or absent).
4. **Rate switching.** For a playlist mixing rates and families (44.1 → 96 → 88.2 → 48 → DSD64 → 44.1), each track satisfies 1–3 with **its own** rate, and the transitions contain only silence.
5. **Unity volume.** With the engine volume at 0 dB, criteria 1–4 hold. With any other volume the engine must **report itself as not bit-perfect** (status API/UI flag), and the check verifies that the flag matches reality.
6. **Negative control.** The same vector routed through a known-lossy path fails criterion 1. If it does not, the run is invalid.

### 3.2 The software verification chain in the virtual T1 (L2, now)

```text
                                                    virtual T1 (Blackbird OS guest)
 host (macOS)                    ┌───────────────────────────────────────────────────────────────┐
 opendap verification check      │  test vector files ──► playback engine ──► hw:Loopback,0,N     │
   audio.bitperfect ──SSH──────► │   (FLAC/WAV/ALAC/DSF/DFF)    (exclusive ALSA)       │ snd-aloop │
   (orchestrates, compares)      │                                                    ▼           │
        ▲                        │  dap-verify capture: hw:Loopback,1,N ──► raw .bin + hw_params │
        │◄──scp results──────────│  snapshot of /proc/asound/Loopback/pcm0p/subN/{hw_params,status}│
        │                        └───────────────────────────────────────────────────────────────┘
  compare(reference decode, capture) → JSON report (requirement IDs, PASS/FAIL, first diff offset)
```

Steps:

1. **Load the loopback** with enough substreams: `modprobe snd-aloop pcm_substreams=2` (module parameter name per `aloop.c`; Debian builds `SND_ALOOP=m`, **UNVERIFIED** for the 7.2 backports config, to be checked first). Ship this as part of a **development system extension** (ADR blackbird-os-0004 already reserves dev extensions), never in the release image.
2. **Start capture first** at the expected format and rate. On `snd-aloop` the two ends are linked: once one end is open, the other must match its parameters. Opening capture first with the *expected* parameters turns a wrong engine choice into an `EINVAL` on the playback side, which is itself a useful assertion (behaviour from `aloop.c` constraints; **UNVERIFIED** in detail).
   - For per-track rate switching, the capture side must follow the playback side instead. The harness then opens capture *after* the engine opens playback, reads `hw_params` of the playback substream and opens capture with the same values.
3. **Capture the raw bytes** with a small purpose-built tool (`dap-verify capture`, Rust + `alsa` crate), not `arecord`. `arecord` writes WAV headers and cannot write DSD containers cleanly (**UNVERIFIED** for alsa-utils 1.2.16), and a dedicated tool can snapshot `hw_params` at each rate change and record xruns.
4. **Reference decode on the host.** For PCM: decode the source with FFmpeg to raw PCM in the same container (`-f s24le`, `s32le`, …) and hash it. FFmpeg 8 has the `hash`, `streamhash` and `framemd5` muxers ([FFmpeg formats](https://ffmpeg.org/ffmpeg-formats.html)) and a `null` muxer, and declares `dsf` and `iff` (DSDIFF) demuxers ([`allformats.c`](https://raw.githubusercontent.com/FFmpeg/FFmpeg/master/libavformat/allformats.c)). For DSD the reference must be the **raw 1-bit stream**, not a decode: FFmpeg's DSD path is built to convert DSD to PCM, so OpenDAP needs its own small DSF/DFF reader (§3.5).
5. **Compare** with a purpose-built comparator: locate the reference's first 4,096 bytes in the capture (the vectors carry a unique sync preamble, §3.3), then compare byte by byte; report `first_mismatch_offset`, `mismatch_count`, `leading_silence`, `trailing_silence`, and the format/rate/channel observed vs expected.
6. **Record** the result (§6.3).

What this proves: the engine (and the decoder it links) delivers the file's samples unchanged, at the right rate and format, through a `hw:` device, with no plug, dmix, PipeWire graph, software volume or dither in the way. What it does not prove: anything below the ALSA PCM boundary (I2S, clocks, DAC).

### 3.3 Test vectors (to build; recommendation)

Generated deterministically by a Rust crate (`opendap-testvectors`) from a seed, so any machine regenerates the same files and hashes. Each vector carries a 64-sample **sync preamble** (a fixed pseudo-random sequence) and an **end marker**, both in-band and full-scale, so the comparator can align without trusting timestamps.

| Vector | Content | What it catches |
|---|---|---|
| `prbs-full` | Uniform pseudo-random samples over the whole code range, every bit toggling, independent L/R seeds | Truncation, dither, gain ≠ 1, F32 round-trip of 32-bit, channel swap |
| `bitwalk` | One bit set per sample, walking LSB → MSB, alternating sign | Any single bit forced or dropped; byte-order errors in 3-byte containers |
| `extremes` | Runs of max positive, max negative, zero, ±1 LSB | Clipping, limiter, DC offset correction, "denormal" handling |
| `sine-997` | 997 Hz at −0.1 dBFS and −60 dBFS, then digital silence | Volume ramps, fades at start/stop, the silence-padding rule |
| `ramp16` | Every 16-bit code in order (65,536 samples per channel) | Lookup or remapping bugs in 16-bit paths |
| `channel-id` | Channel *n* carries the constant pattern 0x5A5A… xor *n* | Channel mapping/remapping (5.1 later, stereo now) |
| `low8-nonzero` (32-bit) | Random 32-bit samples whose low 8 bits are never all zero | The PipeWire F32 graph (negative control) |
| `jtest` | J-test: Fs/4 tone with an LSB square wave at Fs/192 (Dunn's J-test, **UNVERIFIED** exact definition to cite) | Reused later on hardware for jitter (§4) |
| DSD `dsd-prbs` | Pseudo-random 1-bit stream (not musical, the DAC does not need to like it in L2) | Any bit corruption, bit-order reversal |
| DSD `dsd-silence-edge` | 0x69 silence, then data, then 0x69 | DoP marker continuity across silence; mute/unmute transitions |
| DSD `dsd-sine` | A 1 kHz sigma-delta modulated tone generated by a simple 5th-order modulator | Hardware-level sanity later (it actually sounds/measures as a tone) |

**Matrix (L2, per format):** rates {44.1, 48, 88.2, 96, 176.4, 192, 352.8, 384} kHz × containers {S16_LE, S24_3LE, S24_LE (in 32), S32_LE} × file formats {WAV, FLAC, ALAC (**UNVERIFIED** which formats the engine will support)} × channels {2}; DSD {DSD64, DSD128, DSD256} × {DSF, DFF} × output {native `DSD_U8`, `DSD_U16_LE/BE`, `DSD_U32_LE/BE`, DoP over `S24_3LE` and `S32_LE`}. FLAC at 32 bits is used only if the chosen libFLAC supports it (**UNVERIFIED**: FLAC ≥ 1.4 is believed to support 32-bit PCM). Rates above 384 kHz are capped by the DAC choice, not by the loopback.

Third-party "bit-perfect test files" exist (for example files whose decoded content says "pass" only when untouched, as used by HDCD/DTS-detect style tests); they are useful for **human spot checks** on hardware but cannot replace generated vectors, because their licences and content are not under OpenDAP's control (**UNVERIFIED** licences). **Recommendation:** use only generated vectors in the automated suite.

### 3.4 Constrained fake device for rate families and capabilities (to build)

`snd-aloop` accepts any rate, so it cannot show what the engine does when the device **refuses** a rate, or when the 44.1k family is only available after an MCLK switch (BRD-023). Two options, in order of preference (recommendation):

1. **An ALSA `ioplug` PCM plugin** (`libasound_module_pcm_opendap_fake.so`, in `blackbird-os` dev repo) that enforces a **capability table** taken from the DAC datasheet and the board DT (allowed rates per MCLK, formats, DSD support yes/no), simulates the MCLK switch latency, and writes accepted samples to a file. The engine is pointed at it by device name in tests. It is user-space, so OS-002 (no custom kernel) is respected.
2. **Opening `snd-aloop` capture first with a fixed rate** to force refusals (§3.2 step 2). Cheaper, but it can model only one rate at a time.

Pass criteria added by the fake: the engine never asks for a rate outside the table; when the source rate is not supported, it **refuses or reports "not bit-perfect"** rather than resampling silently; when the family changes, it closes and reopens the PCM (observable as a new `hw_params`), and a configurable "settle" gap is respected.

### 3.5 DSD and DoP validators (to build)

**`dsd-src` reader (Rust):** DSF and DFF (DSDIFF) to a canonical in-memory form: per channel, a bit vector in **time order**.

- DSF: per channel 4096-byte blocks, interleaved per block; bits **LSB first** within each byte; last block zero-filled, so the reader must truncate to the declared sample count (spec annotation 3 in the [DSF spec](https://dsd-guide.com/sites/default/files/white-papers/DSFFileFormatSpec_E.pdf)).
- DFF: DSDIFF, MSB first, byte-interleaved channels (**UNVERIFIED** in this session; check against the Philips DSDIFF 1.5 spec before implementing).

#### `dop-check` (Rust, also exposed as `opendap verification check audio.dop` on capture files)

1. Accept S24_3LE or S32_LE frames (for S32 the DoP word is the top 24 bits; the low byte must be 0x00, **recommendation**: report non-zero low bytes as a warning, since some DACs ignore them).
2. For every frame: all channels carry the same marker; markers alternate **0x05 ↔ 0xFA** from one frame to the next ([DoP v1.1 §2](https://dsd-guide.com/sites/default/files/white-papers/DoP_openStandard_1v1.pdf)). Count `marker_violations` and `marker_breaks` (a run interrupted by non-DoP PCM).
3. Extract the 16 payload bits, oldest bit (`t0`) in the most significant payload position, and append them to the channel's bit vector.
4. Check the PCM rate in `hw_params` = DSD rate ÷ 16 (176.4 kHz for DSD64, 352.8 kHz for DSD128 per the spec's §3 option 1). DSD256 at 705.6 kHz follows the same rule but is not in the v1.1 text (**UNVERIFIED** as a standard; widely used).
5. Optionally flag the spec's alternative 0x06/0xF9 marker scheme for 128FS on links limited to 176.4 kHz (spec §3 option 2) as "not used by OpenDAP" so it fails loudly instead of silently.
6. Compare the extracted bit vectors with `dsd-src`, aligned on the `dsd-prbs` preamble.

Pass: `marker_violations == 0` inside the DoP run, `marker_breaks == 0` for a single track, payload identical, rate correct.

**`dsd-native-check`:** the same comparison for `DSD_U8`, `DSD_U16_LE/BE`, `DSD_U32_LE/BE`. Byte order inside the container follows the format name; bit order inside each byte is MSB = oldest (**UNVERIFIED**: confirm against `snd-usb-audio` DSD quirk code or an ALSA developer before relying on it). Silence filling is checked against 0x69.

### 3.6 PipeWire vs exclusive ALSA (what the suite asserts)

| Path | Expected result | Why (source) |
|---|---|---|
| Engine → `hw:` exclusive (aloop) | PASS for all PCM containers, native DSD and DoP | No conversion in the path |
| Engine as PipeWire client, unity volume, `resample.disable`, matching rate, alone | PASS for S16 and S24; **FAIL for S32** `low8-nonzero`; **FAIL for DoP** | "The graph processing format is always float 32" ([pipewire-props(7)](https://docs.pipewire.org/page_man_pipewire-props_7.html)); a float-32 mantissa holds 24 bits |
| Same, with a notification sound mixed in | FAIL during the overlap | Mixing |
| Same, with `node.force-rate` ≠ track rate and `resample.disable` | Link refused | "The node will only be able to negotiate with the graph when the samplerates are compatible" (same source) |
| Engine with volume at −1 dB | FAIL on payload; engine flag says "not bit-perfect" | Criterion 5 |

These rows double as the **negative controls** of §3.1.6. PipeWire's dither modes are "only useful for conversion to a format with less than 24 bits and will be disabled otherwise" (same source), which the S16 row checks indirectly.

### 3.7 Real-hardware verification later (L3/L4)

| What | Method | Instrument |
|---|---|---|
| Bits on the I2S bus (PCM) and DSD lines | Capture BCLK/LRCK/SDATA (and DSD clock/data in DSD mode) at test pads; decode with sigrok's I2S decoder; compare with the same comparator | Logic analyser with ≥ 100 MS/s on 3–4 channels. 384 kHz × 32 bit × 2 ch = **24.576 MHz BCLK**; DSD256 = 11.29 MHz bit clock. sigrok support per model is **UNVERIFIED** |
| Bits to a USB DAC | Capture the isochronous OUT transfers on the T1 itself with `usbmon` and compare payloads; no external instrument | Software only (kernel `usbmon`, **UNVERIFIED** enabled in Debian arm64 config) |
| T1 as USB sound card (UAC2 gadget, BRD-034) | Host plays vectors to the T1; T1 captures and compares | Software only |
| Clock families exact (BRD-023) | Frequency counter / scope on MCLK test pad: 45.1584 MHz and 49.152 MHz within the oscillator's ppm; LRCK = rate exactly MCLK/N | Scope with frequency counter, or a counter with a TCXO/GPSDO reference |
| DAC actually in DSD mode | Read the DAC's status/mode register over I2C (**UNVERIFIED** that the CS43198 exposes a readable DSD-detect status) and measure the analog output of `dsd-sine` (it must be a 1 kHz tone, not −34 dB 88 kHz noise, per DoP v1.1 §2) | Audio analyser |
| End-to-end analog consistency | Play `sine-997` at 0 dBFS, measure level; compare 16/24/32-bit and DSD versions of the same tone | Audio analyser |

A **USB audio interface** on its own cannot prove bit-perfection of an analog output (the ADC adds noise and gain error). It can (a) receive S/PDIF from a USB DAC with a digital output and capture it bit-exactly, which verifies the T1's USB-host path end-to-end; (b) act as an L3 analog analyser for coarse metrics (§4).

---

## 4. P4–P5: analog audio quality and the speaker path

**What "verified" means:** the numbers OpenDAP publishes (THD+N, dynamic range, noise, crosstalk, output impedance, frequency response, output power into 16/32/300 Ω, jitter sidebands) are measured with a declared method (AES17-style filters, declared bandwidth and load) and stay within the published budget. No numeric targets are confirmed yet; the budget is an **open item for the maintainers**.

### Now (L1)

- **Analysis code first, verified on synthetic signals.** Write the THD+N, dynamic range, crosstalk and J-test analysers in Python (numpy/scipy) and test them on generated signals with *known* distortion (a 1 kHz tone plus a −100 dB 2nd harmonic must read −100 dB ± 0.5 dB). This makes the analysis itself verified before any hardware, and the same scripts are reused on the bench (gap analysis §2.7 recommends exactly this).
- **ngspice** (47 on the dev host, with WAV read/write) models of the output stage: OPA1622 output impedance and load behaviour, balanced vs single-ended crosstalk through the shared ground of the TRRS/Pentaconn paths, the speaker amp's isolation from the DAC rails (BRD-012). Vendor models: TI's OPA1622 model availability in a form ngspice runs (not encrypted PSpice) is **UNVERIFIED**.
- **Budget sheet** from datasheets (CS43198 DNR/THD+N, OPA1622 noise, resistor noise) with an estimated system result; marked "datasheet-derived".
- **BRD-012 (speaker independent of DAC):** static check on the netlist (L0) that the speaker amplifier's input does not come from the DAC's analog outputs and has its own enable; L2 check that Blackbird OS exposes the speaker as a separate ALSA PCM or a separate DAI link (with a fake DT under QEMU this is not reachable, so L2 is limited to the UCM/WirePlumber routing config being consistent).

### Later (L3/L4)

- QA403 (US$599; 8 input ranges; up to +18 dBV single-ended / +24 dBV balanced output; REST API) for THD+N, noise, crosstalk, frequency response ([QA403](https://www.quantasylum.com/products/qa403-audio-analyzer)). Residual THD+N spec: **UNVERIFIED** (in the product brief PDF, not read).
- A **load box** with 16 Ω, 32 Ω, 300 Ω and open, switchable, for output impedance (two-load method: Zout = RL · (V_open/V_load − 1)) and power. QuantAsylum's QA451 programmable load (US$319, sold out on 2026-10-21) or a DIY relay box.
- **Jitter:** J-test vector (§3.3) played into the analyser; inspect sidebands around Fs/4 with a long FFT. Clock phase noise directly on MCLK needs a phase-noise analyser; **rent or skip** (recommendation).
- A third-party lab with an Audio Precision APx-class analyser for published numbers (price and model specs **UNVERIFIED**: the vendor site did not resolve in this session).
- **E1DA Cosmos ADC** is a commonly used low-cost, very low-distortion USB ADC for this job (specs and price **UNVERIFIED**: the sources did not load in this session).

---

## 5. Remaining promises (P6–P24)

Each block: **Pass** (what verified means) · **Now** (L0–L2) · **Later** (L3–L5) · **Tools** (existing / build). Automation notes are in §6.

### 5.1 P6 Hearing safety (SYS-012, Draft)

- **Pass:** for any listening history, the computed dose equals the EN 50332-3 Calculated Sound Dose within a stated tolerance; the device warns at 100 % and every further 100 % (per the compliance research, §3; details **UNVERIFIED** against the standard text); the rolling window is seven days; per-headphone profiles map digital level → dB(A).
- **Now (L0/L2), fully software:**
  - **Golden vectors for the dose library** (pure function, unit tests): 100 % CSD = **1.6 Pa²·h**. Check: 80 dB(A) ⇒ p = 20 µPa × 10^(80/20) = 0.2 Pa, p² = 0.04 Pa², × 40 h = 1.6 Pa²·h ✓. Further cases: 83 dB(A) for 20 h = 100 %; 94 dB(A) (1 Pa) for 1.6 h = 100 %; 85 dB(A) for 12.65 h ≈ 100 %.
  - Window edge cases: exposure exactly 7 days ago, clock changes, time-zone changes, reboot persistence, storage corruption (fail safe = assume high dose), headphone hot-swap between 3.5 mm and 4.4 mm (balanced = +6 dB voltage for the same digital level), Bluetooth (out of scope or reported).
  - **A-weighting filter** checked against IEC 61672-1 table values at octave centres (**UNVERIFIED** tolerance class to adopt).
  - **L2 end-to-end in the virtual T1:** play `sine-997` at known dBFS through the engine for simulated hours (accelerated clock injected into the dose service), and assert the warning fires at the expected time.
- **Later:** measure the output voltage per volume step into 32 Ω with the QA403 to build the electrical side of the profiles; the acoustic side (IEC 60318-4 ear simulator) at a lab. EN 50332-2 pink noise (IEC 60268-1 programme simulation noise) as the stimulus.
- **Tools:** existing: Python/numpy, QA403. Build: `dose` crate with test vectors (in `system`), a `hearing-safety` check in the toolkit.

### 5.2 P7 Battery life and power management (T1-009, T1-012, BRD-008, BRD-009, BRD-020 Draft)

- **Pass:** published autonomy for named scenarios (for example "FLAC 44.1/16, screen off, 3.5 mm, 1 mW into 32 Ω") is met within ±10 % on hardware; every block listed by BRD-008 has a measured off-state current; low-power mode (BRD-009) reaches its target state and resumes with all services healthy. No numeric autonomy target is confirmed yet (**open item**).
- **Now:**
  - **Power-model calculator (to build, `board/sim/power/` + `opendap power check`):** YAML of rails and blocks (SoC, LPDDR, eMMC, DAC ×2, amps ×4, display, touch, Wi-Fi/BT, RP2350, PMIC quiescent, fuel gauge), each with states and currents from datasheets (citing the page), plus converter efficiencies; scenarios as state timelines; output energy per hour and hours for the cell (T1-012 ≥ 2,000 mAh) with derating. Unit-tested on hand-computed cases. Every number carries its source, so a model change is reviewable.
  - **Software proxies in the virtual T1 (L2):** for each scenario, measure what software controls and the hardware does not change: CPU time per second of audio per format, wakeups per second (`/proc/interrupts` deltas, timer stats), ALSA period/buffer sizes chosen by the engine (the image-composition research recommends 100–500 ms buffers), services that poll. Assert budgets on them (for example "playback FLAC 44.1: ≤ N wakeups/s"). HVF cores are faster than A55 cores, so CPU **time** is only comparable as a ratio between builds, not in absolute terms.
  - **Suspend/resume (L2):** `s2idle` ("freeze") inside the guest, wake through QMP (`system_wakeup` exists, [QMP reference](https://www.qemu.org/docs/master/interop/qemu-qmp-ref.html)) or an input event; assert that the engine, bus and UI come back and that audio reopens. Whether arm64 `virt` supports deep suspend under HVF is **UNVERIFIED** (likely not); deep suspend on the RK3566 itself is a known pending item.
  - **Battery emulation:** since `test_power` is not built by Debian, put the battery reader behind an interface and test the services against a **sysfs-root fixture** (a directory tree mimicking `/sys/class/power_supply/…`) whose values a test script changes over time (capacity, voltage, status, current). Option for the maintainers: build `test_power` as an out-of-tree module for the dev extension only (it is not a custom kernel, but it is a kernel build; OS-002/OS-006 tension, so **maintainers' decision**).
- **Later:** **L3** dev-kit SoC power per scenario with a Joulescope JS220 (±3 A, 2 Msps, 300 kHz bandwidth, US$999) or PPK2 (≤ 1 A, so it may clip T1 peaks with the display and Wi-Fi on; **UNVERIFIED** T1 peak current); **L4** full rundown tests at constant scenario, logging fuel-gauge (`power_supply` sysfs) every minute and comparing with the integrated energy from the Joulescope; block-by-block off-state current via shunt points (§9). Fuel-gauge driver: the `bq27xxx` family is mainline; BQ27Z561 coverage is **UNVERIFIED**.
- **Tools:** existing: Joulescope (pyjoulescope; licence **UNVERIFIED**), PPK2 (nRF Connect Power Profiler; licence **UNVERIFIED**), PyBaMM 26.10.1.1 for cell models (gap analysis §1.5). Build: power model, scenario runner, sysfs fixtures, rundown logger.

### 5.3 P8 Protection and monitoring (BRD-007, BRD-010, BRD-026 Draft, HCS-015 Draft)

- **Pass:** each protection (over-current, over-voltage, reverse, thermal, battery) trips at its designed threshold and recovers as designed; CPU load and temperatures are exposed and logged; each HAT rail eFuse trips within its limit without disturbing the core.
- **Now:** ngspice simulation of the eFuse and power path where vendor models run in ngspice (**UNVERIFIED** for TPS25947/TPS2121: TI often ships encrypted PSpice models); L0 netlist rule "every HAT power pin passes through an eFuse"; L2 monitoring service tested against a sysfs fixture for `thermal_zone*` and `hwmon*` (the `virt` machine has no SoC thermal zones).
- **Later:** programmable electronic load and supply to sweep currents and record trip time; thermal camera and thermocouple vs reported temperatures; short-circuit tests on the HAT pogo rails.
- **Tools:** existing: ngspice 47, sigrok-cli (Brewfile), SCPI via `pyvisa` for supplies/loads. Build: protection test scripts, netlist rule checks.

### 5.4 P9 Mainline drivers, kernel version, no custom kernel

- **Pass:** for every part with a Linux driver (DAC, PMIC, charger, fuel gauge, touch, panel, ALS, IMU, Wi-Fi/BT, USB-PD, RTC…), the **pinned Debian kernel package** contains the driver and the DT compatible matches; the image kernel is ≥ 7; no kernel is built.
- **Now (L0, high value, cheap):** `opendap driver check` (to build): input = the board DT (and overlays) plus a parts list; for each `compatible`, look it up in `modules.alias` (`of:N*T*C<compatible>*`) and `modules.builtin` of the kernel `.deb` named in the image manifest (`linux-image-7.2.9+deb13-arm64` today); also check the `CONFIG_` symbol in `/boot/config-*` from the package. Output per part: `builtin | module | missing`. The same check runs against Debian `sid` to see what is coming. OS-001 is the trivial check "manifest kernel version ≥ 7".
- **L2:** `modprobe` every listed module in the virtual T1 to catch missing firmware or dependency failures (the hardware is absent, so probe failures are expected; load failures are not).
- **Later:** L3/L4 probe success (`dmesg`, `/sys/bus/*/drivers/*`), and `dt-validate` on the merged DT (co-design research §3.2).
- **Tools:** existing: `dtc`, `fdtoverlay`, dt-schema 2026.9 (co-design §3.2). Build: `driver check`.

### 5.5 P10 Image build, signing, release, flashing

- **Pass:** two builds from the same pinned inputs produce identical images (reproducibility); every release artefact verifies against the OpenDAP key; the verity root hash in the signed FIT matches the erofs root; the default image contains only the declared packages (OS-021); `dap flash` writes and verifies an image.
- **Now:** `opendap image check` (to build): rebuild-and-diff (`diffoscope`), signature verification (FIT: `fit_check_sign` from U-Boot tools, **UNVERIFIED** name in current U-Boot; RAUC: `rauc info` / `rauc verify`), `veritysetup verify` of the root partition against the root hash ([dm-verity docs](https://docs.kernel.org/admin-guide/device-mapper/verity.html)), manifest diff against an allow-list with a size budget, and the existing `build/security-check.py` (package freshness against trixie-security/updates/backports). Tamper tests: flip one byte in the root image and expect verification failure.
- **Later:** `dap flash` against a dev-kit SBC in MaskROM via `rockusb` (lifecycle research finding 3), with read-back verification.
- **Tools:** existing: mkosi, diffoscope, cryptsetup `veritysetup`, RAUC, U-Boot tools, `security-check.py`, SBOM tooling from the compliance research §9.2. Build: `image check`.

### 5.6 P11 Updates, OTA, A/B rollback, recovery

- **Pass (rows of the matrix):** good update → new slot booted → health gate passes → `mark-good`; bad update (health gate fails, kernel panic, watchdog) → try counter runs out → previous slot; power loss during install (QMP `quit`/`system_reset` mid-write) → old slot intact; bundle with a wrong signature → refused; corrupted root block → dm-verity error behaviour (`restart_on_corruption`/`panic_on_corruption`, [dm-verity docs](https://docs.kernel.org/admin-guide/device-mapper/verity.html)) → fallback; downgrade attempt → policy enforced; MaskROM recovery works from a fully erased device.
- **Now (L2):** the **fidelity path**: U-Boot `qemu_arm64` as `-bios` with a pflash environment ([U-Boot docs](https://docs.u-boot.org/en/latest/board/emulation/qemu-arm.html)), the same `boot.scr`/FIT and `bootmeth_rauc` (`BOOT_ORDER`, `BOOT_x_LEFT`) as the T1, RAUC (1.15.2 per [releases](https://github.com/rauc/rauc/releases); release date shown as 2025-03-27 alongside a 2026 CVE, so the date is **UNVERIFIED**). Under HVF this path is **UNVERIFIED** (TCG works per the lifecycle research). Each row above becomes a scripted test: `opendap update check --scenario bad-health-gate`.
- **Later:** the same scenarios on L3/L4 with a USB-controlled power switch for power-loss tests (labgrid 26.0.1 supports power switching, co-design §3.3); MaskROM recovery with the power button (BRD-035) and `rockusb`.
- **Tools:** existing: RAUC, U-Boot, QEMU, labgrid. Build: scenario scripts, health-gate fault injection hooks.

### 5.7 P12 Boot without the RP firmware; boot time

- **Pass:** with the RP2350 erased or held in reset, the SoC boots to a working UI (BRD-019); boot time budget (no number confirmed) is tracked per release.
- **Now:** in L2, boot time per stage from `systemd-analyze` and console timestamps, as a **regression budget only** (HVF ≠ A55). In L0, a DT/service check that no boot-critical unit depends on the coprocessor service.
- **Later:** L4 with the RP2350 held in reset by SWD; wall-clock boot from power-button press (GPIO captured by logic analyser) to the first frame (backlight-enable line or a photodiode).

### 5.8 P13 Virtual T1 fidelity and remote control (OS-009, OS-016, OS-018, OS-019)

- **Pass:** the virtual T1 exposes a 480 × 800 portrait display, multitouch, the four buttons with the documented key codes, audio output, and (when available) battery, coprocessor and HAT presence; it is controllable over SSH and QMP (screenshot, input, scripted flows).
- **Now:** a **self-test of the virtual T1**: `opendap vm check` boots it and asserts each feature (DRM mode 480×800 via `modetest` or sysfs; input devices present with the expected capabilities via `evtest --query`/`libinput list-devices`; QMP `press action` produces `KEY_PLAYPAUSE` in the guest; a `tap` produces ABS_MT events at the expected pixel; `virtio-snd` PCM present). This guards OS-018 against image regressions.
- **Gaps to close (recommendations):** battery (fixture, §5.2), coprocessor (a USB-passthrough Pico 2, §5.9; QEMU `usb-host` passthrough on macOS is **UNVERIFIED**), HAT presence (an emulated EEPROM, §5.12), status LED (`uleds` userspace LED driver, `CONFIG_LEDS_USER`, **UNVERIFIED** in Debian config), ALS/IMU (IIO dummy drivers are **UNVERIFIED** in Debian config; otherwise service-level fixtures).

### 5.9 P14 Peripheral hosting (OS-008, OS-015, BRD-004, BRD-005, BRD-014, BRD-015, HCS-013)

- **Pass:** from Blackbird OS alone, each programmable chip can be identified, flashed, verified, reset, debugged (halt, read registers/memory) and recovered after a failed flash; the RP2350's Try-Before-You-Buy fallback returns to the previous firmware when the new one does not confirm (lifecycle research finding 8); no helper MCU is in any data path (BRD-005).
- **Now:**
  - **No emulator** (finding 7). **Stand-in:** a Pico 2 (RP2350) on the dev host plus a Pico running `debugprobe`. Tests: `picotool load` + `verify`, partition-table A/B, a deliberately non-confirming image to exercise TBYB fallback, SWD halt/flash with OpenOCD (pinned upstream commit with RP2350 and `linuxspidev`/gpiod support, lifecycle research finding 5). These run on macOS through `probe-rs`/`picotool`/`openocd` from the Brewfile; the **same scripts** later run on the T1 against the real coprocessor.
  - **Firmware logic** (RP2350 orchestration code) unit-tested on the host, and its bus protocol with Blackbird OS tested as a contract (golden message files).
  - **BRD-005** as an L0 netlist rule: no net of the audio, display, HAT high-speed or USB data paths has the RP2350 as a driver/receiver.
- **Later:** L3 on a dev-kit SBC with the Pico wired to its GPIO/SPI (SoC-driven SWD as on the T1); L4 on the board. For HATs with FPGAs, openFPGALoader with a gpiod JTAG cable (lifecycle research finding 7).
- **Tools:** existing: picotool, debugprobe, OpenOCD (pinned), probe-rs 0.32.0, openFPGALoader 1.2.0. Build: `opendap chip check` scenario runner (identify / flash / verify / rollback / debug).

### 5.10 P15 Faceplate HID command set (T1-018, T1-020, T1-023, T1-024, FPK-009, FPC-009, ADR t1-0002)

- **Pass:** for each faceplate profile (touch, classic, cassette), the HID report descriptor parses in mainline `hid-input`; each usage in the core set produces exactly the Linux event the spec lists; vendor-page usages reach `hidraw` and not `input`; the vendor/product/version identify the faceplate; malformed descriptors and reports are rejected without crashing.
- **Now (L2), to build: `faceplate-emu`** (Rust, `blackbird-os` dev repo) on `/dev/uhid`:
  1. Generate the descriptor from the spec table (single source: a YAML of the core set in `t1/docs/specs/`, from which both the Markdown table and the golden descriptor are generated).
  2. `UHID_CREATE2` with `bus = BUS_I2C`, the faceplate's VID/PID/version, and the descriptor ([`uhid.h`](https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/linux/uhid.h)).
  3. For every usage: send `UHID_INPUT2` press/release; read the resulting evdev events with `libevdev`; assert the event code from the spec. This turns the spec's "checked 2026-10-21 against `hid-input.c`" into a **regression test against the image's kernel**.
  4. Vendor page: assert the report appears on `hidraw` and Blackbird OS services decode it.
  5. Fuzz: random descriptors and reports (cargo-fuzz on the host for the firmware encoder; in the guest, the kernel's own robustness is not OpenDAP's to test).
  - **hid-tools** 0.12 (2025-11-20, GPL-2.0-only) gives `hid-decode`, `hid-recorder`, `hid-replay` and a UHID-based kernel test suite ([PyPI](https://pypi.org/project/hid-tools/)); use `hid-decode` to review descriptors and `hid-replay` to replay recordings from real faceplates later.
  - **i2c-hid transport** (register map, reset, interrupt line, power during suspend) is not covered by UHID. Verify the faceplate firmware's HID-over-I2C implementation with host-side tests against the Microsoft HID-over-I2C protocol spec (**UNVERIFIED** current version/URL).
- **Later:** L3 with the faceplate controller on a dev board wired to the SBC's I2C (real `i2c-hid` + DT node); logic-analyser capture of the I2C transactions decoded with sigrok (`i2c` decoder; an `i2c-hid` decoder is **UNVERIFIED**); `hid-recorder` on the real device to create replay fixtures.

### 5.11 P16 Display, touch, LED, ALS, microphones, IMU, buttons

- **Pass:** display shows correct content at 480 × 800 with no clipping; touch maps to pixels within a tolerance; every button produces its event (and the Action button is accessible to developers, BRD-036); the status LED is a standard `/sys/class/leds` device a developer can drive (FPT-016, FPK-006, FPC-006); ALS dims the backlight; microphones capture; the privacy LED **cannot** be lit while the microphone is unpowered and vice versa (BRD-027).
- **Now:** L2 via QMP: `screendump` + golden-image comparison with a perceptual tolerance; `tap` at known coordinates and read ABS_MT events; `press` each button. Multitouch gestures need multiple slots in `input-send-event` (**UNVERIFIED** in QEMU 11.1's schema). Microphone path: a test signal written into `snd-aloop` stands in for the mic; the capture pipeline is checked bit-exactly as in §3. **BRD-027 at L0:** netlist rule "privacy LED supply net == microphone supply net, and no GPIO drives the LED". **T1-014/015/022** layout: L0 check on the case export that button features sit on the edges named by the requirement.
- **Later:** L3/L4 display colour/brightness with a colorimeter (optional), touch accuracy with a stylus jig, ALS with a lux meter, microphone sensitivity/noise with a calibrated source, IMU with a turntable or simply gravity at six orientations; BRD-027 measured by toggling mic power and observing the LED.

### 5.12 P17–P18 HAT interface and HAT identity

#### P17 signal integrity (HCS-004/005/006/014, BRD-006, BRD-013, BRD-033)

- **Pass:** each lane meets the channel budget of its standard (insertion/return loss, crosstalk, eye mask) for the rates the RK3566 exposes (PCIe Gen2, USB 3 at 5 Gb/s, MIPI CSI D-PHY; exact lane rates **UNVERIFIED** here); on hardware, links train at the maximum rate and run error-free under load; every compromise has a written justification (HCS-005).
- **Now (L1):** per-segment Touchstone models (board launch, flat cable, device-side piece, pogo region, HAT pads), as the component-export spec already requires; cascade with scikit-rf 2.2.0 and evaluate against budget masks; eye/BER with PyBERT where IBIS-AMI models exist (gap analysis §2.3, §3). **To build:** `opendap channel check` that reads the exports' Touchstone files and a budget YAML, cascades them and reports per-lane margins. **L0:** BRD-006 netlist rule "high-speed nets go SoC → (optional mux) → flat-cable connector with no other active part".
- **Later:** rented ≥ 20 GHz VNA/TDR for the flat cable + pogo assembly; functional: `lspci -vv` LnkSta speed/width, `lsusb -t` at 5000M, `fio` on an NVMe HAT, AER error counters; pogo contact resistance with a 4-wire milliohm meter across insertion cycles.

#### P18 HAT EEPROM and signed manifest (HCS-007…011, HCS-017 Draft)

- **Pass:** a valid EEPROM image is parsed (Raspberry Pi HAT+ format + OpenDAP record), the Ed25519 signature verifies, the manifest hash matches, the reverse-domain ID resolves through the package mechanism; every malformed case is rejected with a clear error: bad signature, wrong hash, truncated, wrong length fields, unknown key, oversize, replayed manifest from another ID.
- **Now (L0/L2):** reference encoder/decoder + **test-vector corpus** in `hat-standard` (to build), fuzzed with cargo-fuzz; L2 in the virtual T1 by presenting the image to the verifier as an nvmem file (the `virt` machine has no I2C bus; the kernel `i2c-stub` module could stand in, **UNVERIFIED** in Debian config). Note ADR blackbird-os-0005 trusts every HAT *for now*: the check must still **report** signature status even when policy accepts.
- **Later:** real EEPROMs on HAT dev boards read by the SoC; Raspberry Pi's EEPROM utilities for cross-checking the HAT+ part (**UNVERIFIED** current tool names in `raspberrypi/utils`).

### 5.13 P19 Connectivity and storage (BRD-029, BRD-030, BRD-034, BRD-038, OS-022)

- **Now:** USB gadget roles (UAC2, serial, network, mass storage) in L2 with `dummy_hcd` (`CONFIG_USB_DUMMY_HCD`, **UNVERIFIED** in Debian config); SD/TF caching (OS-022) with QEMU block throttling on a second virtio disk set to SD-like throughput/latency (`throttling.*` drive options, **UNVERIFIED** exact names in QEMU 11), asserting no audio underrun and correct cache promotion; Wi-Fi/BT driver presence via `driver check` (§5.4).
- **Later:** iperf3 throughput, BT codec negotiation (LDAC/aptX/LC3), AirTag-style tracking (BRD-030) range tests; USB-PD with a Twinkie v2 + sigrok (compliance research §9.3); SD card speed with `fio`.

### 5.14 P20 Mechanics (T1-013, T1-017, T1-021, T1-022, CASE-*, HCS-012/019/020)

- **Pass:** exported assembly fits (no interference, clearances ≥ declared), footprint ≈ ID-1 85.60 × 53.98 mm (tolerance of "approximately" is **not defined**: owner item), coordinate convention respected, Action button travel and click force in a declared range and clearly different from the others (T1-022), snap-fit back cover and screwed parts survive declared cycles, IP65 for the Commercial edition (Draft), drop survival (no requirement number yet).
- **Now:** L0 FreeCAD headless checks on `export/` STEP (bounding box vs ID-1, origin at device centre, +z towards the faceplate, interference between components); 1D tolerance stacks in Python for button-cap-to-switch gap and gasket compression (gap analysis §1.2); FreeCAD FEM/CalculiX 2.23 for snap-fit strain and screw-boss stress; button travel/force from the switch datasheet plus the cap's stack; OpenCourant (young fork) for drop ranking only; IP65 **cannot** be simulated (gap analysis §1.7).
- **Later:** force-displacement curves with a force gauge on a motorised stand (or a load cell + linear stage DIY rig); cycle tests; DIY IPX5 spray pre-check (6.3 mm nozzle) and lab IP65; IEC 60068-2-31-style drops.

### 5.15 P21 Co-design consistency (T1-007, KLIB-002, component-export spec)

- **Pass:** one machine-readable source per interface (pinout YAML), and every derived artefact (KiCad connector netlist, DT fragment, firmware pin map, faceplate/HAT spec tables) agrees with it; every KiCad library design rule has a test board that triggers it (KLIB-002).
- **Now (L0, pre-commit-able):** `opendap interface check`: parse the pinout YAML, `kicad-cli sch export netlist` of each side, the DT/overlays (compiled with `dtc`, applied with `fdtoverlay`, validated with `dt-validate`), and compare names/pins/directions; `kicad-cli pcb drc` on each design-rule test board expecting **exactly** the listed violations (inverting the existing `kicad-drc` hook).

### 5.16 P22 System software (SYS-001…011 Draft, DS-001)

Architecture is not defined yet. **Recommendation:** each requirement becomes a contract test once the bus API exists: manifest schema validation for capabilities (SYS-002), activity lifecycle tests in L2 driven through QMP input and screenshots (SYS-006/007), package fetch only through the package endpoint (SYS-011: in L2, block direct egress with nftables and assert installs still work through the package endpoint and fail without it), and ui-components conformance by screenshot comparison against reference components (DS-001).

### 5.17 P23 Toolkit (TK-001…011, T1-025)

`cargo test` with CLI integration tests (`assert_cmd`-style) on macOS; JSON outputs validated against the published schemas; exit codes per the CLI conventions; the install script tested in a throw-away `$HOME` for install/pin/uninstall/non-interactive (TK-005…007); `shellcheck` on the script; `opendap doctor` checks verification prerequisites too (§6.2).

### 5.18 P24 HAT products

Each HAT repository owns a verification plan in the same format (§6.3). Examples: PWR-005 output current/voltage regulation and efficiency with an electronic load; NAS-005/006 `fio` and `iperf3`; VNA-005 calibration against SOLT standards with scikit-rf; RAD-005 against HackRF reference measurements; REC-005 reuses the §3 bit-perfect harness in the capture direction; OSC-005 bandwidth/rise time with a reference generator. All are later (L3+).

---

## 6. Proposed instrumentation architecture (recommendation)

### 6.1 Where things live

| Repository | Holds | Examples |
|---|---|---|
| `toolkit` | Host-side orchestration and analysers; the report schema; the traceability generator | `crates/opendap` verification commands; `crates/opendap-verify-model` (schema); `crates/opendap-testvectors`; `crates/opendap-audiocheck` (PCM comparator, `dop-check`, `dsd-native-check`, DSF/DFF reader); `crates/opendap-dose` reused from `system` or vice versa; Python analysis for THD+N/J-test under `tools/analysis/` with `uv` |
| `blackbird-os` | Guest-side helpers in the **development repository** (OS-015), the QEMU profiles and the fidelity path | `dap-verify` package (capture harness, `hw_params` snapshots, `faceplate-emu`, fixtures); dev system extension that loads `snd-aloop`; `build/dap-qemu.sh --fidelity` (U-Boot); scenario scripts for update/rollback |
| `system` | Unit and contract tests of the services | Playback engine (exclusive ALSA, rate switching, bit-perfect flag), dose service, battery/thermal readers with fixtures |
| `t1` | Product verification plan and matrix; faceplate interface source YAML and golden HID descriptors | `docs/verification/` (plan + generated status), `docs/specs/faceplate-interface.yaml` |
| `board` | Simulations and bring-up tests | `sim/power/` model, `sim/audio/` ngspice, `sim/si/` openEMS/scikit-rf, `test/bringup/` checklists, netlist rules |
| `hat-standard` | Manifest schema, reference implementation, test vectors, channel budget | `vectors/eeprom/`, `sim/channel/` |
| `case`, `faceplates/*` | Mechanical studies and checks | FEM studies, tolerance stacks, STEP checks |
| `kicad-lib` | Design-rule test boards (KLIB-002) | Expected-violation files |
| `claude-plugins` | Skills that wrap the commands for agents | `/opendap:verify` (run a check, record, explain), `/opendap:trace` (refresh matrix) |

### 6.2 CLI surface (recommendation, aligned with the CLI conventions)

The conventions use `noun verb` with a closed verb set that includes `check`, `list`, `get`. Proposal:

| Command | Purpose | Target |
|---|---|---|
| `opendap verification list [--requirement BRD-023] [--level L2]` | List checks, their requirement IDs and levels | — |
| `opendap verification check <check-id>… [--target vm:default\|usb:…\|net:…]` | Run checks; NDJSON progress; final JSON report | VM, device or host |
| `opendap verification get <run-id>` | Read a stored result | — |
| `opendap requirement get <ID>` | Promise, checks, latest results, highest level reached | — |
| `opendap requirement list --status unverified` | What is not covered | — |
| Alias `opendap verify <area> [--bitperfect …]` | Documented shortcut mapping to `verification check` (as in the brief) | — |

Initial check IDs: `audio.bitperfect.pcm`, `audio.bitperfect.dsd-native`, `audio.bitperfect.dop`, `audio.rate-switch`, `audio.negative-control`, `audio.analog` (needs analyser), `hearing.dose`, `power.model`, `power.proxies`, `power.suspend`, `driver.mainline`, `image.reproducible`, `image.signature`, `image.verity`, `update.rollback.*`, `vm.self-test`, `faceplate.hid.<profile>`, `hat.manifest`, `hat.channel`, `interface.consistency`, `chip.rp2350.*`, `mech.envelope`, `mech.stack.*`, `kicad.rules`.

`opendap doctor` gains checks for verification prerequisites: QEMU, the fidelity firmware, the dev extension, FFmpeg, `uv`, a Pico 2 present, an analyser reachable over HTTP, a Joulescope present.

### 6.3 Results and traceability

**Result record** (JSON, `"schema":"opendap.verification-result/v1"`), one per check run:

```json
{"schema":"opendap.verification-result/v1",
 "check":"audio.bitperfect.dop","requirements":["OS-020","BRD-022"],
 "level":"L2","target":{"kind":"vm","name":"default","image":"blackbird-os-qemu 2026.10.22+abc123"},
 "tools":{"opendap":"0.1.0","dop-check":"0.1.0","ffmpeg":"8.1.1","kernel":"7.2.9+deb13-arm64"},
 "inputs":[{"vector":"dsd-prbs-dsd64.dsf","sha256":"…"}],
 "observed":{"hw_params":{"format":"S24_3LE","rate":176400,"channels":2},
             "marker_violations":0,"payload_mismatch_bits":0},
 "negative_control":{"check":"audio.negative-control.dop-pipewire","result":"fail-as-expected"},
 "result":"pass","started":"…","duration_s":42,
 "evidence":["results/2026-10-22T10-00Z/audio.bitperfect.dop/capture.bin.zst"]}
```

- **Storage (no CI):** results under `.state/verification/` in the workspace (not committed); a **release** attaches the result set (and evidence hashes) as an artefact and commits a regenerated `docs/verification/status.md` in each repository. This matches the existing local-release model.
- **Traceability:** checks declare their requirement IDs in code (attribute or registry), the co-design research's linking conventions apply to non-code artefacts, and `opendap requirement list` builds the matrix from the requirement READMEs + check registry + latest results. A requirement with no check is reported as **unverified**, never silently green.
- **Automation tiers** (no CI):
  - **pre-commit (seconds):** L0 checks on changed files: `interface.consistency`, netlist rules, `kicad.rules` (existing hooks), schema validation of manifests, dose and power-model unit tests, `driver.mainline` when the DT or parts list changes.
  - **on demand / pre-release (minutes to an hour):** L2 suites against the virtual T1 (`audio.*`, `faceplate.*`, `vm.self-test`, `update.*`, `image.*`).
  - **bench sessions (L3/L4):** the same commands with `--target usb:…`, plus instrument-backed checks; results recorded the same way.

---

## 7. Phased plan (recommendation)

| Phase | When | Deliverables |
|---|---|---|
| **Now (L0–L2)** | Before any hardware | 1. Verify the L2 assumptions in §8. 2. `opendap-testvectors`, `opendap-audiocheck` (PCM comparator, `dop-check`, DSD reader/check), `dap-verify capture`, aloop dev extension → `audio.*` checks with negative controls. 3. `faceplate-emu` + core-set YAML. 4. Fidelity QEMU path + `update.rollback.*`. 5. `driver.mainline`, `image.*`, `interface.consistency`, BRD-027/BRD-005/BRD-006 netlist rules. 6. Power model + proxies + suspend test. 7. Dose library + vectors. 8. Report schema and `requirement list`. 9. Pico 2 stand-in scripts for `chip.rp2350.*`. 10. Synthetic-signal-verified THD+N/J-test analysers. 11. Design-for-test items in the board (§9). |
| **Dev-kit hardware (L3)** | When an RK3566 SBC, DAC eval board, fuel-gauge eval and the Wi-Fi module eval are on the bench | `audio.*` on real ALSA drivers (I2S captured with a logic analyser), MCLK family switching, SoC power per scenario (Joulescope), real `i2c-hid` faceplate controller, SoC-driven SWD to a Pico 2, `dap flash` over MaskROM, analog metrics on the DAC eval board |
| **T1 prototypes (L4)** | EVT/DVT | Full matrix on the product: rundowns, protections, HAT channel (rented VNA/TDR), link training at max rate, mechanics (force curves, drops, spray), thermal, hearing-safety electrical profiles, EMC pre-scan |
| **Certification (L5)** | Commercial edition | Lab EMC/RED, 62368-1, IP65, EN 50332 acoustic; evidence linked to the same requirement IDs |

---

## 8. First checks to run in the virtual T1 (all currently UNVERIFIED)

1. `snd-aloop` loads on the 7.2 backports kernel and `aplay -l`/`/proc/asound/cards` lists `Loopback`.
2. Playback opened with `DSD_U32_BE` at 88.2 kHz, `S24_3LE` at 352.8 kHz and `S32_LE` at 384 kHz succeeds on the loopback, and `hw_params` reports them.
3. `status` exposes the owning PID.
4. `/dev/uhid` exists and a `BUS_I2C` device with the core-set descriptor produces `KEY_PLAYPAUSE` etc.
5. QMP `input-send-event` supports multitouch slots in QEMU 11.1.
6. `s2idle` suspend and resume in the guest, woken by QMP.
7. U-Boot `qemu_arm64` boots under HVF (otherwise TCG for the fidelity path).
8. Debian config symbols: `SND_ALOOP`, `USB_DUMMY_HCD`, `LEDS_USER`, `IIO_SIMPLE_DUMMY`, `I2C_STUB`, `USB_MON`.

---

## 9. Design decisions affected now (recommendations)

1. **Audio test pads** on MCLK, BCLK, LRCK, SDATA (and DSD clock/data) of each DAC, reachable with the faceplate removed. Without them, bit-perfection below ALSA (§3.7) cannot be measured on the T1.
2. **Current-measurement points:** a sense resistor or a removable link on the battery path and on each major rail (SoC, DDR, audio analog, display, radio, HAT), so BRD-008/T1-009 can be measured per block.
3. **RP2350 SWD and BOOTSEL** exposed as pads (in addition to the SoC-driven SWD), so a bench probe can recover it independently of Blackbird OS.
4. **Faceplate I2C + INT** and **HAT I2C (EEPROM)** reachable for a logic analyser.
5. **HAT high-speed lanes:** keep a probe-friendly launch (or coupons on the panel) so a VNA/TDR can characterise board + flat cable separately from the pogo region.
6. **A test-mode DT overlay** that exposes debug GPIOs (stage markers for boot-time measurement).
7. **Engine API:** the playback engine reports a machine-readable "bit-perfect: yes/no + reason" status. It is what §3.1 criterion 5 verifies and what users see.
8. **Battery and thermal readers** take a sysfs root path (fixture-friendly).

---

## 10. Minimum lab instruments (recommendation)

| Instrument | Purpose | When | Price class |
|---|---|---|---|
| Raspberry Pi Pico 2 ×2 (one as `debugprobe`) | RP2350 stand-in and SWD probe | Now | < US$20 total (**UNVERIFIED** current price) |
| Logic analyser, ≥ 100 MS/s, ≥ 8 ch, sigrok-supported | I2S/DSD capture, I2C, boot markers | Dev kit | US$100–400 (**UNVERIFIED**) |
| QuantAsylum QA403 | THD+N, noise, crosstalk, FR, output impedance, hearing-safety voltages | Dev kit | **US$599** ([source](https://quantasylum.com/collections/frontpage)); needs a Windows (or Linux/Mono) host |
| Load box 16/32/300 Ω (QA451 US$319, sold out; or DIY) | Output impedance and power | Dev kit | US$50–320 |
| Joulescope JS220 | Power profiling, rundown cross-check | Dev kit | **US$999** ([source](https://www.joulescope.com/products/js220-joulescope-precision-energy-analyzer)) |
| Nordic PPK2 | Low-current states (sleep, off), as a source for single-cell voltage | Dev kit | ~US$100 (**UNVERIFIED**) |
| Programmable bench supply + electronic load (SCPI) | Protections, eFuse trips, charger tests | Dev kit | US$300–800 (**UNVERIFIED**) |
| Oscilloscope ≥ 200 MHz with counter | MCLK frequency, power sequencing, boot timing | Dev kit | US$400–1,000 (**UNVERIFIED**) |
| Thermal camera (USB) | Hot spots, BRD-010 cross-check | T1 prototype | US$250–400 (**UNVERIFIED**) |
| Force gauge + motorised stand (or load cell rig) | T1-022 click force/travel | T1 prototype | US$100 DIY – US$1,500 (**UNVERIFIED**) |
| 4-wire milliohm meter | Pogo contact resistance | T1 prototype | US$100–300 (**UNVERIFIED**) |
| tinySA Ultra + near-field probes | EMC pre-scan | T1 prototype | US$150–300 (**UNVERIFIED**) |
| Twinkie v2 | USB-PD analysis | T1 prototype | ~US$50–100 (**UNVERIFIED**) |
| ≥ 20 GHz VNA / TDR | HAT channel (PCIe/USB3 over pogo) | T1 prototype | **Rent** or lab service |
| Audio Precision APx-class, ear simulator | Published audio numbers, EN 50332 acoustic | Pre-launch | **Lab service** |

---

## 11. Traceability table

Legend: **Now** = software/simulation/static verification available before hardware; **Later** = measurement on hardware; **Tool** = E existing / B to build. Levels as in §1.1.

| Req ID | Status | Promise | Software verification now | Hardware measurement later | Tool |
|---|---|---|---|---|---|
| OS-020 | Confirmed | Bit-perfect PCM, native DSD and DoP | L2 aloop capture vs generated vectors; `hw_params` oracle; DoP marker/payload check; native DSD check; PipeWire negative controls | I2S/DSD lines via logic analyser; `usbmon` for USB DACs; analog `dsd-sine` check | B: testvectors, audiocheck, dap-verify capture, dop-check; E: snd-aloop, FFmpeg 8, sigrok |
| BRD-023 | Draft | Two MCLK families, DAC master, no resampling at any rate | L2 constrained fake PCM (ioplug) with the clock table; rate-switch playlist | Counter on MCLK (45.1584/49.152 MHz, ppm); LRCK = MCLK/N; per-track `hw_params` on device | B: ioplug fake; E: scope/counter |
| BRD-022 | Draft | 2× CS43198 dual mono + 4× OPA1622; DSD capable | L0 netlist; L1 ngspice output stage; L2 DSD formats exposed by the codec driver (needs the real driver, so L3) | Analyser per channel; DSD mode readback | E: ngspice, QA403 |
| T1-008 | Confirmed | Audio quality top priority | Synthetic-verified analysers; datasheet budget | THD+N, DNR, crosstalk, Zout, FR, J-test | B: analysis scripts; E: QA403 |
| BRD-011 | Confirmed | 3.5 mm and 4.4 mm outputs | L0 netlist/STEP: both jacks present | Per-jack measurements | E |
| BRD-024 | Draft | 3.5 SE, 4.4 balanced (Pentaconn) | L0 netlist topology rule | Balanced vs SE level (+6 dB), crosstalk | B: rule; E: QA403 |
| BRD-012 | Confirmed | Speaker independent of DAC | L0 netlist rule; L2 routing config consistency | Speaker plays with DAC powered down | B |
| SYS-012 | Draft | Sound dose 7-day, 85 dB(A), per-headphone | Golden dose vectors; window edge cases; L2 accelerated-clock warning test | Voltage per volume step (QA403); lab acoustic | B: dose crate |
| T1-009 | Confirmed | Max battery life | Power model; L2 wakeups/CPU proxies; s2idle test | Joulescope scenarios, rundowns | B: model, proxies; E: Joulescope |
| T1-012 | Confirmed | Standard cell ≥ 2,000 mAh | L0 BOM/case check of cell part and capacity | Capacity test (discharge at C/5) | E: e-load |
| BRD-008 | Confirmed | Power down unused blocks | L0 every block has an enable/rail switch; L2 runtime-PM policy tests on fixtures | Per-rail off current via sense points | B |
| BRD-009 | Confirmed | Low-power mode | L2 s2idle enter/resume health | Suspend current, resume latency | E: PPK2/Joulescope |
| BRD-020 | Draft | RK809 PMIC | `driver.mainline` for `rk809` | Rail sequencing on scope | B |
| BRD-007 | Confirmed | Protection and safety | ngspice (where models run); netlist rules | Trip tests with supply/load | E/B |
| BRD-010 | Confirmed | Monitor CPU and temperature | Service tests on sysfs fixtures | Thermocouple/thermal camera vs reported | B |
| BRD-026 | Draft | eFuse per HAT rail | L0 netlist rule; ngspice | Short-circuit trip on each rail | B |
| HCS-015 | Draft | Power classes A/B/F | Schema + manifest vectors for classes; power-path sim | Source/sink tests with powerbank HAT | B |
| BRD-001 | Confirmed | Mainline drivers only | `driver.mainline` on pinned Debian kernel | Probe success on L3/L4 | B |
| OS-001 | Confirmed | Kernel ≥ 7 | Manifest check | — | B (trivial) |
| OS-002, OS-006 | Confirmed | No custom kernel / embedded builds | L0: image build log contains no kernel/toolchain compile; kernel from Debian archive | — | B |
| OS-007 | Draft | Kernel updatable as prebuilt package | L2: install newer backports kernel package in dev mode / via image update; boots | Same on device | E: apt, RAUC |
| FPT-008 | Draft | GT911 touch | `driver.mainline` (`goodix`) | Touch accuracy jig | B |
| FPT-018 | Confirmed | Panel with mainline driver (or OpenDAP adds it) | `driver.mainline` on the panel compatible | Panel init on L3 | B |
| BRD-025 | Draft | Pre-certified Wi-Fi/BT module | `driver.mainline` for module; certificate IDs recorded | iperf3, BT codecs | B/E |
| OS-004, OS-005, OS-010, OS-021 | Confirmed | Trimmed image from code and prebuilt packages; minimal default | Manifest allow-list + size budget; rebuild diff | Image fits/boots on device | B: image check; E: mkosi, diffoscope |
| OS-011 | Confirmed | Signed, versioned releases | Signature/verity verification + tamper tests | — | E: RAUC, veritysetup; B |
| OS-012 | Confirmed | Simple flashing | `dap flash --dry-run` plan tests | MaskROM flash + read-back on SBC/T1 | B (`rockusb`) |
| OS-013 | Confirmed | Updates from Debian and OpenDAP repos | `security-check.py`; APT repo snapshot test | — | E |
| OS-014 | Draft | OTA | L2 RAUC streaming install from local HTTP | Over Wi-Fi on device | E: RAUC |
| OS-017 | Draft | MaskROM recovery; automatic A/B rollback | L2 fidelity path scenarios (bad gate, power loss, verity corruption, bad signature) | Same with power switch; MaskROM recovery | B: scenarios; E: U-Boot, RAUC, labgrid |
| BRD-016, BRD-035 | Confirmed | MaskROM via main/power button | L0 netlist (button to boot-mode strap) | Enter MaskROM and flash | B/E |
| BRD-017 | Confirmed | Recovery key | L0 netlist; U-Boot script test in fidelity path (key → recovery) | Physical test | B |
| BRD-018, BRD-034 | Confirmed | USB-C OTG, flashing, host, charging… | L2 gadget roles with `dummy_hcd` (UNVERIFIED) | Each role on device; PD with Twinkie | E/B |
| BRD-039 | Confirmed | Early console on the USB-C path | — (hardware path) | Console capture during boot | E |
| BRD-019 | Confirmed | Boot without RP firmware | L0 no boot-critical dependency on coprocessor service | Boot with RP2350 held in reset | B |
| OS-009, OS-016 | Confirmed | Run Blackbird OS on host (QEMU), headless automation | `vm.self-test` | — | E: dap-qemu.sh |
| OS-018 | Confirmed | Virtual T1 mimics hardware | `vm.self-test` (display, touch, buttons, audio; battery/RP/HAT gaps listed) | Compare with device behaviour | B |
| OS-019 | Confirmed | Remote control (desktop, SSH, screenshots, input) | QMP screenshot/press/tap round-trip tests | Same over USB/net on device | E: dap-qmp.py; B |
| OS-008, HCS-013 | Confirmed | Blackbird OS flashes/operates/debugs every chip | Pico 2 stand-in: load/verify/TBYB fallback/SWD | SoC-driven SWD on T1 and HATs | E: picotool, debugprobe, OpenOCD; B: chip check |
| OS-015 | Confirmed | Dev tools in a separate repo, tested on device | L2 install OpenOCD from dev repo, run `--version` and a self-test | Same on device | B |
| BRD-004 | Confirmed | SoC reaches every chip directly | L0 netlist: every programmable chip's debug/boot pins reach SoC GPIO/bus | Flash each chip from SoC | B |
| BRD-005 | Confirmed | Helper MCU never in a data path | L0 netlist rule | — | B |
| BRD-014, BRD-015 | Confirmed | RP coprocessor, flashable from Blackbird OS | Pico 2 stand-in | On board | E/B |
| T1-023, T1-024, FPK-009, FPC-009 | Confirmed | Standardized binary commands (HID core + vendor page) | `faceplate-emu` over UHID: every usage → expected event; vendor page → hidraw | Real controller over `i2c-hid`; I2C capture | B: faceplate-emu; E: hid-tools 0.12 |
| T1-020 | Confirmed | Identification by interface (HID VID/PID/version) | UHID test with each profile's IDs | Real faceplates | B |
| T1-018 | Confirmed | Faceplate interface defined | L0 spec completeness check (pinout, dimensions, anchoring present) | — | B |
| FPT-002, FPT-013 | Confirmed | 3.5" IPS touch display | L2 480×800 mode + touch mapping | Panel/touch on device | E/B |
| FPT-003, FPT-016, FPK-006, FPC-006 | Confirmed | Developer-controllable status LED | L2 `uleds` (UNVERIFIED) LED class test | LED on device | B |
| FPT-004, FPT-017, BRD-031 | Confirmed | Microphones (faceplate main, ambient top right) | L2 capture path bit-exact via aloop | Sensitivity/noise | B/E |
| FPT-010 | Draft | ALS + mic next to LED | L2 service fixtures (IIO dummy UNVERIFIED) | Lux meter | B |
| FPT-014, FPK-004, FPC-004 | Confirmed | Display + ALS dimming | L2 dimming logic on fixtures | Lux sweep vs backlight | B |
| FPT-015, FPK-005, FPC-005 | Confirmed | Display driver on the faceplate | L0 netlist split check | — | B |
| FPK-007/008, FPC-007/008 | Confirmed (should) | OLED, physical controls | Descriptor profiles for classic/cassette in `faceplate-emu` | Real faceplates | B |
| BRD-027 | Draft | Privacy LED wired to mic power | L0 netlist rule | Toggle mic power, observe LED | B |
| BRD-032 | Confirmed | IMU | `driver.mainline`; service fixtures | Six-orientation gravity test | B |
| BRD-036, T1-014, T1-015 | Confirmed | Buttons and layout; Action accessible | L2 QMP press → events; L0 case export feature positions | Physical test | E/B |
| T1-022 | Confirmed | Action button deeper travel, clearer click | Tolerance stack + switch datasheet | Force-displacement curves | B; E: force gauge |
| HCS-004, HCS-005 | Confirmed | Max speed per lane; compromises justified | Touchstone cascade vs budget; justification doc check | VNA/TDR; link speed/width; AER | B: channel check; E: scikit-rf, PyBERT |
| HCS-006, HCS-014 | Confirmed / Draft | One interconnect (pogo), passive HAT pads | L0 spec/STEP check | Contact resistance over cycles | B |
| BRD-006 | Confirmed | High-speed buses direct to HAT (muxes allowed) | L0 netlist rule | Link training | B |
| BRD-013 | Confirmed | Board owns flat-cable spec | L0 export present with Touchstone | — | B |
| BRD-033 | Confirmed (should) | PCIe on board optional | — | — | — |
| HCS-007, HCS-009, HCS-017 | Confirmed / Draft | EEPROM with HAT+ format + OpenDAP record | Reference parser + vectors + fuzzing | Real EEPROM read by SoC | B |
| HCS-010 | Confirmed | Manifest signed and verified | Signature vectors (valid/invalid/tampered) | — | B |
| HCS-008, HCS-011 | Confirmed | No central registry; reverse-domain via the package endpoint | L2 resolve ID through local package-endpoint stand-in | — | B |
| BRD-029, BRD-030 | Confirmed | Wi-Fi/BT; item tracking | `driver.mainline`; protocol tests on host BT (UNVERIFIED feasibility) | iperf3, range tests | B/E |
| BRD-038, OS-022 | Confirmed | SD/TF; caching in eMMC/RAM | L2 throttled virtio disk, underrun and cache tests | `fio` on real SD | B |
| BRD-037 | Draft | 4 GB RAM / 32 GB storage | L2 runs at 4 GB/32 GB (already the VM size); memory budget test | — | E |
| T1-013 | Confirmed | ~ID-1 footprint | L0 STEP bounding box | Calipers | B |
| T1-021 | Draft | Coordinate system, ISO 2768-m | L0 export origin/axes check | CMM/calipers on parts | B |
| T1-017 | Draft | IP65 (Commercial) | Gasket compression FEM; no OSS ingress check | Spray pre-check; lab IP65 | E: CalculiX; lab |
| CASE-006, HCS-012, HCS-020, CASE-024 | Confirmed | Swappable back/HAT case; screwed parts | L0 assembly interference; boss FEM | Fit and cycle tests | E/B |
| CASE-025 | Confirmed | Snap-on plain back | Snap-fit strain FEM | Cycle test | E |
| CASE-020, CASE-021, HCS-019 | Confirmed | Polymer, graphite grip finish, metal lock allowed | L0 material fields in exports | Sample inspection | B |
| T1-007 | Confirmed | Assembled and simulated from exports | L0 `component.yaml` + assembly build from exports only | — | B |
| KLIB-002 | Draft | Design rules with triggering test boards | DRC expects listed violations | — | E: kicad-cli; B |
| SYS-001…SYS-011 | Draft | Capabilities, bus, activities, registry, packages via the package endpoint | Contract tests once APIs exist; L2 egress-blocked package-endpoint test | — | B |
| DS-001 | Confirmed | UI follows design system | Screenshot comparison in L2 | — | B |
| TK-001…TK-011, T1-025 | Confirmed | Toolkit CLI properties | Cargo tests, schema/exit-code conformance, install-script tests in temp HOME | — | E/B |
| T1-001, BLIB-002 | Confirmed | Open source; licences | `reuse lint` (pre-commit) | — | E: reuse |
| T1-002, T1-003, BRD-003, HCS-001…003 | Confirmed | General-purpose gadget; HATs and faceplates | Covered by P14–P18 | — | — |
| T1-004…006, T1-010, BRD-002, BRD-028 | Confirmed | Editions, sourcing, cost, single PCB | L0 BOM availability/price script (distributor APIs, UNVERIFIED access) | — | B |
| T1-019 | Confirmed | Faceplates not swapped regularly | Informs flex/connector cycle rating (no test) | — | — |
| KLIB-001, FLIB-001, BLIB-001 | Confirmed | Shared libraries | Library lint (existing skills) | — | E |
| CAM-005, NAS-005/006, OSC-005, PWR-005, RAD-005, REC-005, VNA-005 (+ *-001…004) | Confirmed / Draft | HAT products | Per-HAT plans; REC reuses the bit-perfect harness | Per-HAT bench tests | B |

---

## 12. Open questions for the maintainers

1. Numeric targets: audio (THD+N, DNR, Zout, power), autonomy per scenario, boot time, footprint tolerance. Without them, "verified" can only mean "measured and recorded".
2. Is an out-of-tree `test_power` (dev extension only) acceptable under OS-002/OS-006?
3. Which DSD rates and formats are promised (DSD64/128/256; native over I2S, over USB, DoP)?
4. Should the release gate require L2 audio checks to pass (recommendation: yes)?
5. Windows/Linux helper host for the QA403, or wait for an open macOS driver?

---

## 13. Sources (accessed 2026-10-21/22)

- Linux `snd-aloop` formats and rates: <https://raw.githubusercontent.com/torvalds/linux/master/sound/drivers/aloop.c>
- Linux DSD silence/physical widths: <https://raw.githubusercontent.com/torvalds/linux/master/sound/core/pcm_misc.c>
- ALSA proc files (`hw_params`, `status`): <https://www.kernel.org/doc/html/latest/sound/designs/procfile.html>
- QEMU virtio-snd formats/rates: <https://raw.githubusercontent.com/qemu/qemu/master/hw/audio/virtio-snd.c>
- QEMU QMP reference: <https://www.qemu.org/docs/master/interop/qemu-qmp-ref.html>
- DoP open standard v1.1: <https://dsd-guide.com/sites/default/files/white-papers/DoP_openStandard_1v1.pdf>
- Sony DSF file format specification: <https://dsd-guide.com/sites/default/files/white-papers/DSFFileFormatSpec_E.pdf>
- PipeWire properties (F32 graph, resample/channelmix/dither, force-rate): <https://docs.pipewire.org/page_man_pipewire-props_7.html>
- FFmpeg formats (hash muxers): <https://ffmpeg.org/ffmpeg-formats.html> ; registry (dsf, iff, null): <https://raw.githubusercontent.com/FFmpeg/FFmpeg/master/libavformat/allformats.c>
- ALSA project releases (alsa-lib/utils 1.2.16, 1.2.16.1): <https://www.alsa-project.org/wiki/Main_Page>
- UHID: <https://www.kernel.org/doc/html/latest/hid/uhid.html> ; header: <https://raw.githubusercontent.com/torvalds/linux/master/include/uapi/linux/uhid.h>
- hid-tools 0.12: <https://pypi.org/project/hid-tools/>
- test_power parameters: <https://raw.githubusercontent.com/torvalds/linux/master/drivers/power/supply/test_power.c>
- U-Boot on QEMU arm64: <https://docs.u-boot.org/en/latest/board/emulation/qemu-arm.html>
- RAUC releases: <https://github.com/rauc/rauc/releases> ; integration: <https://rauc.readthedocs.io/en/latest/integration.html>
- dm-verity: <https://docs.kernel.org/admin-guide/device-mapper/verity.html>
- Renode releases: <https://github.com/renode/renode/releases>
- rp2040js: <https://github.com/wokwi/rp2040js>
- picotool README: <https://raw.githubusercontent.com/raspberrypi/picotool/master/README.md>
- debugprobe: <https://github.com/raspberrypi/debugprobe>
- QuantAsylum products and QA403: <https://quantasylum.com/collections/frontpage> , <https://www.quantasylum.com/products/qa403-audio-analyzer>
- Joulescope JS220: <https://www.joulescope.com/products/js220-joulescope-precision-energy-analyzer>
- Nordic PPK2: <https://www.nordicsemi.com/Products/Development-hardware/Power-Profiler-Kit-2>
- Sibling OpenDAP research (versions of KiCad, FreeCAD FEM, CalculiX, openEMS, scikit-rf, PyBERT, labgrid, dt-schema, OpenOCD, probe-rs, RAUC/U-Boot bootmeth, compliance): see the links in the header.
