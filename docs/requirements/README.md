# Verification toolkit requirements

Requirements TK-001, TK-009, TK-010 and TK-011 in [toolkit-cli](https://github.com/opendap-project/toolkit-cli/blob/main/docs/requirements/README.md) apply to every toolkit repository.

| ID | Requirement | Priority | Status |
|---|---|---|---|
| TK-012 | Verification shall rely on simulations and on samples captured from sessions, on the virtual T1 now and on physical prototypes later. Dedicated lab instruments are not assumed. | must | Confirmed |
| TK-013 | The verification instrumentation proposed in `docs/research/2026-10-verification-instrumentation.md` is approved, except its lab instruments. | must | Confirmed |
- Audio without instruments: bit-perfect is a property of the digital stream, so it is verified by comparing captured bits. (1) Virtual T1: `snd-aloop` capture of the playback engine output, with negative controls. (2) Prototype: a passive tap of the I2S/DSD lines at test pads, captured by the board's RP2350 (PIO) and streamed to the host; a passive listener is not in the data path. (3) Analog output: recordings with a consumer USB audio interface, or with the recorder HAT's ADC, for relative measurements and regressions, not calibrated figures.
