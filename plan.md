# LED Board — Project Plan

A Raspberry Pi–driven RGB LED matrix display with a browser-based designer for
composing text, emoji, images and animations.

---

## 1. Hardware

### Panels — 3 × 64×64 RGB matrix
Identified from board silkscreen:

| Property | Value |
|---|---|
| Model | `DCHY-P2-6464-1515-VP` (sticker: `P2-8L-200`) |
| Resolution | **64 × 64** per panel |
| Pitch | **P2** (2 mm) → each panel ≈ **128 × 128 mm** |
| Connector | **HUB-75E** (the `E` 5th address line is required for 64-row panels) |
| Driver ICs | CHIPONE `ICN2037BP` / `ICN2038S` |
| Scan rate | 1/32 |

### Controller — Adafruit RGB Matrix Bonnet for Raspberry Pi
- HUB75 output, 5 V screw terminal for external power, `5V OK` indicator LED
- GPIO breakout labelled `OE / CLK / LAT`, `R1 G1 B1 / R2 G2 B2`, address lines `A B C D E`
- Compatible with `hzeller/rpi-rgb-led-matrix` via `--led-gpio-mapping=adafruit-hat`
  (or `adafruit-hat-pwm` after the PWM mod, for less flicker)

### Power
- Each 64×64 P2 panel draws up to **~4 A @ 5 V** at full white (~20 W)
- 3 panels → **~12 A / 60 W peak**
- Requires a dedicated **5 V 10–15 A** supply into the bonnet's screw terminal
- ⚠️ Do **not** power the panels from the Pi's own supply
- Realistic text/emoji content draws far less, but size the PSU for peak

---

## 2. Layout / canvas shapes

**Initial target: `64 × 192`** (64 wide × 192 tall — vertical/portrait stack of 3 panels).

The design must not hard-code this. Planned supported arrangements:

| Shape | Description |
|---|---|
| **64 × 192** | Portrait — 3 panels stacked vertically *(first target)* |
| 192 × 64 | Landscape — 3 panels in a row |
| L-shape | Non-rectangular arrangement |

**Implication:** the renderer needs a **panel-mapping layer** that translates a
logical canvas (any shape) into the physical chain order + per-panel rotation.
`rpi-rgb-led-matrix` supports this via `--led-chain`, `--led-parallel` and its
pixel-mapper (`U-mapper`, `Rotate:90`, etc.); custom shapes (L) may need a
custom mapper or a software-side coordinate remap.

---

## 3. Software stack

- **Driver:** [`hzeller/rpi-rgb-led-matrix`](https://github.com/hzeller/rpi-rgb-led-matrix)
  — de-facto standard for HUB75 on Pi, has Python bindings
- Example invocation (landscape chain of 3):
  ```
  --led-rows=64 --led-cols=64 --led-chain=3 \
  --led-gpio-mapping=adafruit-hat
  ```

---

## 4. ⚠️ Gotchas (address before first power-on)

1. **Solder the E-line jumper on the bonnet.**
   The Adafruit bonnet does not route the `E` address line by default, but
   **64×64 panels require it**. Bridge **GPIO 4 → pin 8 pad**. Without it only
   half the panel lights up. This is the most common failure for 64×64 + Adafruit HAT.

2. **Pi 5 is NOT supported.** The library doesn't work with the Pi 5's RP1 GPIO.
   → **`www` (Pi 5) is ruled out.**

3. **Disable onboard audio:** add `dtparam=audio=off` to `/boot/firmware/config.txt`.
   The onboard sound shares PWM hardware and causes flicker.

4. **Dedicate the machine.** The driver bit-bangs GPIO and is timing-sensitive.
   Best results with an isolated CPU core (`isolcpus=3` on a quad-core).

---

## 5. Host selection

| Host | Model | Verdict |
|---|---|---|
| `www` (192.168.1.112) | **Pi 5** | ❌ library unsupported |
| `pi` (192.168.1.134) | Pi 4B Rev 1.4 | ❌ runs Homebridge / garage door — don't add timing-critical GPIO work |
| `pi2` (192.168.1.113) | Pi 4B Rev 1.2 | ❌ CI host — build bursts would cause flicker |
| **Pi Zero** | **TBD** | ⏳ **pending — Rax to confirm model** |

### Pi Zero feasibility (pending confirmation)
The library bit-bangs GPIO, so refresh rate scales with CPU. 3 chained
64×64 panels = **12,288 pixels**.

| Model | Verdict for 3 panels |
|---|---|
| Pi Zero / Zero W (BCM2835, **single-core ARMv6**) | ⚠️ Marginal — expect flicker / low refresh. Mitigate with `--led-pwm-bits=7`, `--led-slowdown-gpio=0..1` (costs color depth) |
| **Pi Zero 2 W** (BCM2710, **quad-core A53**) | ✅ Workable (~Pi 3 class) with tuning |

> Check with: `cat /proc/cpuinfo | grep -i model` on the Zero.

**Note:** driving the 3 panels as *three independent 64×64 displays* is far
lighter on CPU than one 192-long chain — a viable fallback for a weaker Zero.

**Conclusion:** a **dedicated** board is the right pattern. Pi Zero 2 W, or a
cheap dedicated Pi 3A+/Pi 4.

---

## 6. The app

A **web-based designer with a pixel-accurate preview**, usable with **no hardware attached**.

### Designer / simulator (build first)
- Canvas rendered as individual **LED dots** with accurate color + brightness
- Arbitrary canvas shape (64×192, 192×64, L-shape)
- Tools:
  - **Text** — bitmap fonts that render crisply at small pixel sizes
  - **Emoji**
  - **Image import** — auto-downscale + dithering for low-res output
  - Color picker, per-zone layout
- **Animations:** static, scrolling text, blink, transitions — previewed live
- Save / load designs

### Pi runtime (build second)
- Renders designs to a framebuffer and drives the matrix via `rpi-rgb-led-matrix`
- "Send to Pi" from the designer
- Panel-mapping layer for the chosen physical arrangement

### Why simulator first
It's hardware-independent — design and iterate in the browser while the
soldering, power supply and host selection are still being sorted.

---

## 7. Open items

- [ ] Confirm Pi Zero model (Rax) → decides host + whether chaining or 3 independent panels
- [ ] Solder E-line jumper (GPIO 4 → pin 8) on the bonnet
- [ ] Source 5 V 10–15 A PSU
- [ ] Decide chain wiring order for the 64×192 portrait arrangement
- [ ] Build browser designer/simulator
- [ ] Build Pi runtime + push pipeline
