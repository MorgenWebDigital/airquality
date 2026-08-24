# Design notes

Why the parts are what they are, and what was tried before them. Written from
the board as built, not from the plan. The two diverged along the way.

---

## 1. Sensors

**SCD40 for CO₂.** The only sensor here that measures CO₂ rather than guessing
at it. It works photoacoustically: infrared light is pulsed at CO₂ molecules in
a small chamber, they warm and expand, and a microphone hears the pressure
pulse. Accuracy ±(50 ppm + 5 % of reading).

Its datasheet is unusually specific about the supply: ripple below 30 mV
peak-to-peak, no large transient drops, and a low-dropout regulator preferred.
That is not politeness. The sensor draws 205 mA peaks during a measurement and
18 mA average, and a supply that sags under that peak gives wrong readings.
VDD and VDDH must sit at the same voltage and are tied together close to the
part.

**SGP41 for VOC and NOx.** A heated metal-oxide cell. It reports an *index*,
not a concentration. It reads as "the air is getting stale", and is not
comparable to ppm. It is the only part on the board that heats continuously
while measuring, at about 11 mW.

The plan originally called for an ENS160. It needs a 1.71–1.98 V core supply,
which would have meant a second regulator on a board that otherwise runs on one
rail. The SGP41 runs from 3.3 V and needed no extra hardware.

**SHT41 and T117Z for temperature.** Two sensors on purpose. The SHT41 gives
humidity plus a temperature good to ±0.2 °C. The T117Z is a
TMP117-register-compatible part from Mysentech that reaches ±0.1 °C for a
fraction of the TI original's price. It is the precision channel; the SHT41's
temperature is there mostly to compensate its own humidity reading.

The T117Z's ADD0 pin is tied to ground, which selects address `0x48`.

**BH1750 for light.** Its DVI pin is the I²C reference voltage, not a reset.
Tied to 3.3 V it sets the logic level. ADDR to ground gives `0x23`.

---

## 2. Power

**Why a PD trigger at all.** The CH224K negotiates 9 V over USB Power Delivery
instead of accepting the default 5 V. For the same charging power the input
current halves: less loss in the cable, less heat in the connector, less heat
in the charger. A 6.8 kΩ resistor on CFG1 selects 9 V; the value is the
datasheet's, not a guess. CFG2 and CFG3 must be left floating for the
single-resistor mode, and they are.

9 V is also comfortably inside the BQ25895's 3.9–14 V input range.

**Why a switching charger.** The plan started with an MCP73831, a linear
charger limited to 0.5 A. A 4500 mAh cell at 0.5 A takes nine hours, and the
regulator burns the difference between input and cell voltage as heat. The
BQ25895 switches instead, handles up to 5 A, and can be told what to do over
I²C. The input current limit is set by a 250 Ω resistor to about 2.1 A.

The TS pin has a fixed 10 k/10 k divider from REGN instead of a thermistor,
which puts it at 50 % of REGN, inside the window where the charger considers
the cell temperature normal and charges. Adding a real NTC later needs only the
two resistors swapped.

**Why cell protection is separate.** The Molicel P45B ships unprotected. The
DW01A watches the cell and switches an FS8205A dual MOSFET: over-charge above
4.25 V, over-discharge below 2.4 V, over-current, short circuit. It works
whether or not the charger is running or the microcontroller is awake.

The FS8205A's two drains are internally connected and brought out on pins 2 and
5; both are left unconnected, which is correct. The pair sits in series in the
cell's negative path with the sources at either end.

A 1 kΩ resistor sits between the DW01A's VM pin and the pack negative. The
datasheet's application circuit shows it and the reason is in the absolute
maximum ratings: VM tolerates VDD−24 V to VDD+0.3 V. With the protection
tripped and a charger connected, the full charger voltage lands on that pin,
and without the resistor nothing limits the current into it. Leaving it out is
a known way to kill a DW01A.

**Why the regulator can't be switched off.** The XC6222's CE pin is active
high and tied to SYS, so the regulator runs whenever the cell has charge. It is
tempting to put that pin on a GPIO for a hard power-down, but the ESP32 runs
from the very rail it would be switching, and could never switch it back on.

---

## 3. The e-paper supply

The panel's own controller runs the boost converter: it drives the gate of an
external MOSFET from its GDR pin and senses the current on RESE. All the board
provides is the power stage: a 47 µH inductor, an SI1308 MOSFET, a 2.2 Ω sense
resistor and three Schottky diodes.

Two things about this circuit are easy to get wrong, and both were:

**Diode polarity.** For the boost, the anode must sit at the switching node.
Otherwise the inductor has nowhere to dump its energy when the MOSFET turns
off, VGH never rises, and the MOSFET sees uncontrolled flyback. The same logic
inverts for the charge pump that makes the negative VGL. All three diodes were
built the wrong way round at first. The trap is the symbol: for the MBR0530
used here, **pin 1 is the cathode**, not the anode.

**Capacitor voltage rating.** VGH runs above 20 V and VGL to about −20 V. A
1 µF 0603 from the shelf is frequently rated 6.3 or 16 V. The reference circuit
specifies 25 V for all of them, and the bill of materials now says so
explicitly.

A 25 V X5R at 20 V bias also loses 60–80 % of its nominal capacitance to DC
bias. The panel manufacturer evidently accounts for that; if in doubt, a 50 V
part or an 0805 case gives back the margin.

---

## 4. Layout decisions

**The antenna.** The ESP32 module sits with its antenna at the board edge, and
a keep-out covers that area on both copper layers: no pour, no track, no via.
A ground plane under a module antenna detunes it badly enough to make the radio
useless.

**ESD at the connector.** The USB data lines pass *through* a USBLC6-2SC6, in
on one pin of a channel and out on the other, rather than having the protection
hang off them as a stub. The device sits 1.9 mm from the connector pads, with a
ground via directly at its ground pad.

Both details come from the same arithmetic. An IEC 61000-4-2 pulse at 8 kV has
a first peak near 30 A with a rise time around 0.8 ns. A track over a ground
plane has roughly 1 nH per millimetre. That gives **about 35 V of overshoot per
millimetre** of track between the connector and the clamp. The ground return
counts in the same loop, which is why the via matters as much as the
distance.

**Current and track width.** See [`track-widths.md`](track-widths.md).

---

## 5. What is deliberately not fitted

**U12, the SGP41, is marked do-not-populate** in the current build. Its
footprint, routing and I²C address are all present; fitting it later needs no
board change, only the part. It does not appear in the bill of materials or the
pick-and-place file, which is the point of the marking.
