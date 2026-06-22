# Unity-Gain Voltage Buffer — SkyWater SKY130 (130 nm, 1.8 V)

A closed-loop unity-gain voltage buffer (analog "voltage follower") designed and
simulated in the fully open-source **SkyWater SKY130** 130 nm process at **1.8 V**.

Everything here is reproducible with open tools — `ngspice` and the open
[SKY130 PDK](https://github.com/google/skywater-pdk) (`sky130_fd_pr` device models) —
and contains no proprietary data.

---

## Architecture

![schematic](schematic/sky130_schematic.svg)

| Block | Devices | Function |
|-------|---------|----------|
| **5T OTA** | NMOS diff pair `M1/M2`, PMOS mirror load `M3/M4`, NMOS tail `M5` | single-stage transconductance amplifier |
| **Output stage** | NMOS source follower `M6` + poly resistor `Rdeg` + current sink `M7` | low-impedance output; `Rdeg` (source/emitter degeneration) sets a defined output node and aids stability |
| **Bias** | ideal `Iref` → diode `Mb` → mirror | `Iref` (10 µA) mirrored to the tail (×2) and the output sink (×4) |
| **Compensation** | `Cc = 4 pF` at the OTA output `d2` | dominant-pole compensation |

**Feedback:** the bottom of `Rdeg` (= `VOUT`) drives the current sink and feeds back to
the OTA negative input (`vinn`), closing the unity-gain loop. The input pair and
follower use the **low-Vt** flavor (`nfet_01v8_lvt`) to recover input-common-mode and
output headroom on the 1.8 V rail.

---

## Specifications & simulation results

Operating point: **VDD = 1.8 V, Vout = 0.6 V, CL = 5 pF, 27 °C, typical corner.**
Loop gain measured by series voltage injection (`T = −v(vout)/v(vinn)`); offset 1σ from
a 300-point device-mismatch Monte-Carlo (`tt_mm`).

| Parameter | Spec (target) | Simulated |
|-----------|---------------|-----------|
| Supply voltage | 1.8 V | 1.8 V |
| Quiescent current | < 100 µA | **76 µA** |
| **Power** | < 200 µW | **137 µW** |
| **DC loop gain** | > 30 dB | **36.3 dB** |
| **Phase margin** | > 60° | **66°** |
| Unity-gain frequency | — | **3.66 MHz** |
| Closed-loop bandwidth (−3 dB) | — | **5.72 MHz** |
| **Settling, 0.1%** (100 mV step) | < 400 ns | **249 ns** |
| Overshoot | < 15 % | **7.7 %** |
| Slew rate (up / down) | — | **3.7 / 2.7 V/µs** |
| **Output noise, RMS** (1 Hz–1 GHz) | — | **199 µVrms** |
| Systematic offset @ 0.6 V | — | **−9.1 mV** |
| **Offset 1σ** (MC mismatch) | < 5 mV | **3.5 mV** |
| **Output range** (gain 0.95–1.05) | — | **0.15 – 0.75 V** |
| Active device area Σ(W·L) | — | **84 µm²** (+ 4 pF comp cap) |

### Process-corner results (Vout = 0.6 V)

| Corner | Power | Sys. offset | Loop gain | UGF | Phase margin |
|--------|-------|-------------|-----------|-----|--------------|
| SS | 133 µW | +9.6 mV | 36.1 dB | 3.24 MHz | 68.5° |
| TT | 137 µW | +9.1 mV | 36.3 dB | 3.66 MHz | 66.0° |
| FF | 140 µW | +8.5 mV | 36.5 dB | 4.06 MHz | 63.8° |

Phase margin stays 64–69° and gain ≈ 36 dB across corners → robust.

---

## Design notes

- **Output range is the lower part of the rail (≈ 0.15–0.75 V).** The NMOS input pair
  needs a relatively *high* common mode, while the NMOS source follower can only pull
  the output *up* to about `VDD − Vgs6 − I·Rdeg` (worsened by body effect). The usable
  window is therefore the lower portion of the supply — intrinsic to an
  NMOS-input / NMOS-follower buffer. A deep-nwell follower (bulk = source) or a
  complementary output stage would extend the high side.
- **Accuracy** is set by the single-stage 36 dB loop gain (≈ 1.5 % gain error). A
  cascode/gain-boosted first stage would reduce the offset further.
- **The topology ports across nodes and supplies** — only device flavor, sizing, and the
  compensation cap change. (A 3.3 V variant in a 180 nm node follows the same approach,
  operating over the lower ~half of that rail.)

---

## Reproduce

Requires `ngspice` and the open SKY130 PDK (`volare enable --pdk sky130 <version>`).

```sh
cd spice
# point the .lib path at your PDK install:
sed -i "s#PDK_ROOT#$PDK_ROOT#g" *.spice
ngspice -b tb_op_dc.spice       # operating point, power, DC transfer
ngspice -b tb_loopgain.spice    # loop gain, phase margin, UGF
ngspice -b tb_tran.spice        # step settling / overshoot
ngspice -b tb_noise.spice       # integrated output noise
ngspice -b tb_mc_offset.spice   # 300-pt mismatch Monte-Carlo offset (tt_mm)
```

## Files

```
README.md                       this document
schematic/sky130_schematic.svg   schematic (vector) + .png
spice/vbuffer.spice              parametrized buffer subcircuit
spice/models.spice               PDK model include (edit PDK path)
spice/tb_*.spice                 testbenches (op/dc, loop gain, transient, noise, MC)
```

## License

MIT (see `LICENSE`). Uses the open-source SkyWater SKY130 PDK (Apache-2.0).
