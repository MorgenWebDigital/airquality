# Track widths and current

Copper is 35 µm (1 oz), all tracks on outer layers. Widths follow
**IPC-2221** at a 10 K temperature rise, which is the conservative end of that
standard.

---

## 1. The arithmetic

IPC-2221 gives the cross-section for a permitted temperature rise:

```text
   I = k · ΔT^0.44 · A^0.725        k = 0.048 for outer layers
```

Solved for width at 35 µm copper and ΔT = 10 K:

| Current | Width needed |
|---|---|
| 0.5 A | 0.12 mm |
| 1.0 A | 0.30 mm |
| 2.0 A | 0.78 mm |
| 2.25 A | 0.92 mm |
| 3.0 A | 1.37 mm |
| 4.5 A | 2.39 mm |

Doubling the permitted rise to 20 K cuts these by roughly a third, but 10 K
leaves room for a warm enclosure and for the charger running beside them.

---

## 2. What actually flows

The cell is a Molicel INR21700-P45B, 4500 mAh. Charge rates:

- **0.5 C = 2.25 A**, the intended default. Cooler, gentler on the cell.
- **1 C = 4.5 A** is possible; the BQ25895 handles up to 5 A and can be told to
  over I²C.

On the input side the CH224K requests 9 V, so the same charging power needs
about half the current a 5 V supply would. The input limit is set by a 250 Ω
resistor to roughly 2.1 A.

On the 3.3 V rail the worst case is the ESP32 transmitting (345 mA) while the
SCD40 measures (205 mA), around 560 mA, and only in bursts of milliseconds.

---

## 3. What is on the board

| Net | Carries | Width on the board |
|---|---|---|
| **BAT−** | full cell current, up to 4.5 A | 0.80 – 1.20 mm |
| **VBAT** | full cell current | 0.50 – 1.20 mm |
| **SYS** | system rail from the charger | 0.20 – 1.20 mm |
| **SW** | the charger's switching node | 1.20 mm |
| **VBUS** | input from USB, up to ~2.1 A at 9 V | 0.20 – 0.60 mm |
| **+3V3** | up to ~560 mA in bursts | 0.20 – 0.40 mm |
| **GND** | return, backed by the pour | 0.20 – 0.50 mm |
| VGH, VGL | display gate rails, microamps | 0.20 mm |
| signals | I²C, SPI, USB, buttons | 0.20 mm |

The minima are narrow because a track does not carry its full current over its
whole length. Where a wide track has to reach a pad on a 0.5 mm pitch package,
or squeeze between a via and a pad, it narrows for a millimetre or two and
widens again. The heating over such a short stretch is spread by the copper on
either side.

The **switching node SW is kept at a constant 1.2 mm** and as short as the
placement allows. It is not a current question. It is the node with the
steepest edges on the board, and its loop area sets how much of that ends up
as radiated noise.

**Ground returns through the pour**, not through tracks. The 0.2 mm ground
segments are stubs from pads to vias, not current paths.

---

## 4. The limit that actually bit

Widening tracks is not free. On a dense board the widened copper runs out of
clearance to its neighbours, and the design rule here is 0.2 mm.

Several places needed a compromise: the track was widened over its free run and
kept narrow where it passes a pad or a via. Two spots were caught at under
0.1 mm during review, one of them **VBAT against ground near the cell**. At
that spacing an etching defect bridges the two, which on a lithium cell is not
a cosmetic problem.

The rule of thumb that came out of it: widen the free stretches, leave the
pinch points alone, and let the DRC decide where the boundary is.
