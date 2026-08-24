# Changelog

This file follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and
the versions follow [semantic versioning](https://semver.org/).

## [Unreleased]

### Fixed

- the 3.3 v regulator was held off. its ce pin is active high and sat on
  ground, so the whole board had no supply. ce now goes to sys
- all three schottky diodes of the display supply were reversed. the mbr0530
  symbol has the cathode on pin 1, and the boost needs its anode at the
  switching node. as drawn, vgh could never charge and the mosfet saw
  unclamped flyback
- vgl had no bulk capacitor. the reference circuit calls for 1 µf; the other
  twelve were present, that one was missing
- the dw01a's vm pin sat directly on ground. its datasheet puts 1 kΩ in series,
  because vm sees the full charger voltage when the protection has tripped

### Added

- esd protection on the usb data lines, routed through the device and 1.9 mm
  from the connector, with a ground via at its ground pad
- pull-up on io2. espressif recommends it against glitches on that strapping
  pin
- pin 4 of the sgp41 tied to ground, as its datasheet asks
- voltage rating on the seven display rail capacitors. vgh runs above 20 v and
  a stock 1 µf 0603 is often a 16 v part

### Changed

- qr code on the silkscreen now points at the github account instead of the
  shop. same size, verified by decoding it back out of the footprint file
- track widths on the power nets follow ipc-2221 at a 10 k rise
