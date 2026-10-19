<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="docs/logo/benchport-logo-on-dark.svg">
    <img src="docs/logo/benchport-logo-on-light.svg" alt="Benchport" width="560">
  </picture>
</p>

# Benchport: under-desk USB-C dock and debug station.

| Top | Side | Front |
|:---:|:---:|:---:|
| ![top](https://image.easyeda.com/oshwhub/pullImage/ca96a6eacd824d9591369acb5b3c3872.jpg) | ![side](https://image.easyeda.com/oshwhub/pullImage/35ed8f0b45684cfda32ebeee98f4ef13.jpg) | ![front](https://image.easyeda.com/oshwhub/pullImage/68c121f7d69c444a84d63350737d9a25.jpg) |

An open-source under-desk docking station for electronics benches. It combines a USB 3 hub, USB-C power delivery to your laptop, and a built-in multi-protocol programming and debug interface. One USB-C cable to the laptop gives you charging, USB ports and a programmer/debugger, with no extra dongles.

### Final desk integration (render)

| Without the debug puck | With the debug puck |  
|:---:|:---:|
| ![image.png](https://image.easyeda.com/oshwhub/pullImage/6facc709c2bd49bba8ee3624c85f7682.png)| ![image.png](https://image.easyeda.com/oshwhub/pullImage/d7d1ed7361704d1697551c1bc19356ed.png) |

*Concept render: the port block sits flush in the desk surface, with the magnetic puck dock next to it. The electronics mount underneath.*

## Overview

Hardware work means constantly swapping programmers, serial adapters and debug probes. This dock keeps all of that in one place. A laptop connects with a single USB-C cable, and the desk surface has swappable magnetic accessory pucks for whichever tool you need.

## Features

- **USB ports:**
  - 2 × USB-C, USB 3.0 SuperSpeed
  - 1 × USB-A, USB 3.0 SuperSpeed
  - 1 × USB-A, USB 2.0
  - 1 × USB-C on the bottom of the board, USB 2.0
- **USB 3 hub:** a TI TUSB8041A 4-port USB 3.0 hub, with a WCH CH334P USB 2.0 hub behind it for the USB 2.0 ports and the internal debug bridge.
- **Laptop charging over the same cable:** a TI TPS25751 USB PD controller sources up to 100 W to the laptop (5 V and 20 V PDOs). It then swaps data role so the laptop is the USB host.
- **USB-C power input:** a CH221K PD sink takes 20 V from a USB-C PD charger (tested with a Lenovo 130 W). A TPS548A20 buck makes the 5 V rail.
- **SuperSpeed muxing on USB-C:** the upstream port uses an HD3SS3212 and the main downstream USB-C port uses an HD3SS3220, so both run USB 3 in either plug orientation. The second downstream USB-C port is wired single-sided: SuperSpeed in one orientation, USB 2.0 when flipped.
- **Current-limited ports:** every downstream port has a TPS2552/TPS2553 switch set to ≥ 900 mA, enough for ESP32-type boards during Wi-Fi bursts.
- **Built-in programmer/debugger:** a WCH CH347F on a dedicated internal hub port provides UART, SPI, I2C, JTAG and SWD.
- **Modular magnetic dock:** a pogo-pin interface on the desk surface for swappable 3D-printed pucks (Tag-Connect, SWD, SPI flash clip, JTAG, serial).
- **Automatic mode selection, no MCU:** each puck has an ID resistor on CONFIG_ID. An LM2901B comparator ladder decodes it, and a TS3A27518E analog mux switches the shared pads between an SPI/I2C group and a JTAG/UART group.
- **Safe target power:** a TPS2115A selects 3.3 V or 5 V for VCC_TARGET. Power only turns on after the puck is fully seated and the voltage choice has settled. The level shifter is enabled after VCC_TARGET is up and disabled instantly when the puck is removed.
- **Level shifting:** a TXS0108E sits between the CH347F and the exposed pads.
- **ESD protection** on every external USB line and on the pogo interface.

## Architecture

| Block | Part | Role |
|---|---|---|
| Power input | CH221K | USB-C PD sink, 20 V |
| 5 V rail | TPS548A20 | Synchronous buck, 20 V → 5 V |
| 3.3 V rail | TLV62569 | Buck, 5 V → 3.3 V |
| 1.1 V rail | TPS74611 | LDO for the hub core |
| Laptop PD | TPS25751 + CAT24C512 EEPROM | USB PD source, data-role swap |
| USB 3 hub | TUSB8041A | 4-port USB 3.0 hub |
| USB 2 hub | CH334P | USB 2.0 ports + CH347F |
| SS muxes | HD3SS3212, HD3SS3220 | USB-C orientation switching |
| Port power | TPS2552 / TPS2553 | Current-limited VBUS switches |
| Debug bridge | CH347F | UART / SPI / I2C / JTAG / SWD |
| Mode decode | LM2901B | Comparator ladder for CONFIG_ID |
| Target power | TPS2115A | 3.3 V / 5 V selection for VCC_TARGET |
| Pad mux | TS3A27518E | Switches shared pads between protocol groups |
| Level shifter | TXS0108E | Target-voltage translation |

## Puck ID resistors

The puck's resistor from CONFIG_ID to GND sets the mode and the target voltage. Use 1 % resistors.

| Mode | Rpuck |
|---|---|
| 3.3 V, SPI / I2C | 130 kΩ |
| 5 V, SPI / I2C | 37.4 kΩ |
| 5 V, JTAG / UART | 13.0 kΩ |
| 3.3 V, JTAG / UART | 1.0 kΩ (or 0 Ω) |
| No puck | open, target power off |

I2C pucks should carry their own 4.7 kΩ pull-ups from SDA/SCL to VCC_TARGET.

## ⚠️ Important: flash the PD controller's EEPROM first

**The board won't work until the TPS25751's EEPROM (U9, CAT24C512) has been programmed.**

The TPS25751 boots in SafeMode (ADCIN1/ADCIN2 tied to GND) and loads its firmware and configuration from the EEPROM at address 0x50. With a blank EEPROM, the PD chip stays idle. That means no PD contract and no power to the laptop. It also means no data-role swap, so the hub never connects, and **the computer won't detect the dock at all.**

To set it up:

1. Download **[`config_TPS25751_F8091166_fullFlash.bin`](firmware/tps25751/config_TPS25751_F8091166_fullFlash.bin)** from [`firmware/tps25751/`](firmware/tps25751/). It's a ready-to-flash Full Flash image with the configuration below.
2. Write the binary to the EEPROM. There are two ways:
   - **Through the TPS25751:** connect to the I2Ct test pads (SCL/SDA/GND, I2C address 0x21). Program with the FLad → FLwd → FLvy → GAID 4CC command sequence from the TPS25751 Technical Reference Manual.
   - **Directly:** write the image to the CAT24C512 on the I2Cc bus at 0x50.
3. Power-cycle the board. The TPS25751 loads the image, and the dock should now charge the laptop and enumerate.

Configuration in the provided image (for reference, or to rebuild it in TI's TPS25751 Application Customization Tool):
- Type-C state machine: **Source only**
- Data role: **initiate and accept swap to UFP**; reject swap to DFP and all power-role swaps to sink
- USB Communication Capable, USB3 Gen1
- Source PDOs: **5 V / 3 A** and **20 V / 5 A**
- GPIO1 = UFP/DFP event (push-pull), which drives the hub's USB_VBUS detect
- GPIO2 = cable orientation event (push-pull), which drives the upstream SS mux

## Hardware files

The board was designed in EasyEDA Pro and fabricated and assembled at JLCPCB.

| File | Description |
|---|---|
| [`hardware/easyeda/benchport-mainboard-revA.epro2`](hardware/easyeda/benchport-mainboard-revA.epro2) | Main board EasyEDA Pro project (editable source) |
| [`hardware/easyeda/benchport-pogo-dock-revA.epro2`](hardware/easyeda/benchport-pogo-dock-revA.epro2) | Pogo dock board EasyEDA Pro project (magnetic puck dock, FPC link to the main board) |
| [`hardware/easyeda/benchport-puck-i2c-spi-revA.epro2`](hardware/easyeda/benchport-puck-i2c-spi-revA.epro2) | I2C/SPI puck EasyEDA Pro project |
| [`hardware/mainboard/benchport-mainboard-schematic.pdf`](hardware/mainboard/benchport-mainboard-schematic.pdf) | Main board schematic (PDF) |
| [`hardware/mainboard/fabrication/benchport-mainboard-gerbers-revA.zip`](hardware/mainboard/fabrication/benchport-mainboard-gerbers-revA.zip) | Main board Gerbers and drill files (4-layer), ready to upload to JLCPCB |
| [`hardware/mainboard/fabrication/benchport-mainboard-bom-revA.csv`](hardware/mainboard/fabrication/benchport-mainboard-bom-revA.csv) | Main board BOM with LCSC part numbers |
| [`hardware/mainboard/fabrication/benchport-mainboard-cpl-revA.csv`](hardware/mainboard/fabrication/benchport-mainboard-cpl-revA.csv) | Main board pick-and-place (CPL) file |
| [`hardware/pogo-board/fabrication/benchport-pogo-dock-gerbers-revA.zip`](hardware/pogo-board/fabrication/benchport-pogo-dock-gerbers-revA.zip) | Pogo dock Gerbers and drill files (2-layer), ready to upload to JLCPCB |
| [`hardware/pogo-board/fabrication/benchport-pogo-dock-bom-revA.csv`](hardware/pogo-board/fabrication/benchport-pogo-dock-bom-revA.csv) | Pogo dock BOM with LCSC part numbers |
| [`hardware/pogo-board/fabrication/benchport-pogo-dock-cpl-revA.csv`](hardware/pogo-board/fabrication/benchport-pogo-dock-cpl-revA.csv) | Pogo dock pick-and-place (CPL) file |
| [`hardware/puck-i2c-spi/fabrication/benchport-puck-i2c-spi-gerbers-revA.zip`](hardware/puck-i2c-spi/fabrication/benchport-puck-i2c-spi-gerbers-revA.zip) | I2C/SPI puck Gerbers and drill files (2-layer), ready to upload to JLCPCB |
| [`hardware/puck-i2c-spi/fabrication/benchport-puck-i2c-spi-bom-revA.csv`](hardware/puck-i2c-spi/fabrication/benchport-puck-i2c-spi-bom-revA.csv) | I2C/SPI puck BOM with LCSC part numbers (R1, the ID resistor, is left as TBD; see [Puck ID resistors](#puck-id-resistors)) |
| [`hardware/puck-i2c-spi/fabrication/benchport-puck-i2c-spi-cpl-revA.csv`](hardware/puck-i2c-spi/fabrication/benchport-puck-i2c-spi-cpl-revA.csv) | I2C/SPI puck pick-and-place (CPL) file |
| [`firmware/tps25751/`](firmware/tps25751/) | TPS25751 PD controller EEPROM image and configuration |

Open the `.epro2` files with **File → Open → EasyEDA Pro project** in EasyEDA Pro.

## Design decisions

- **No MCU for mode selection.** A passive resistor ID plus comparators is simpler, cheaper, deterministic, and has no firmware to maintain. The only programmed part on the board is the PD controller's EEPROM.
- **No 1.8 V support.** Only 3.3 V and 5 V are offered, which keeps level shifting and target power simple.
- **Two-state pad muxing.** Pads are shared between two protocol groups instead of giving every protocol its own pins.
- **All parts in stock at LCSC**, for JLCPCB 4-layer PCBA.

## Getting started

1. Flash `config_TPS25751_F8091166_fullFlash.bin` to the TPS25751 EEPROM (see above). This only needs to be done once.
2. Connect a USB-C PD charger (20 V capable) to the power input.
3. Connect the upstream USB-C port to your laptop. It charges and enumerates the hub. For 100 W (20 V / 5 A), use a 5 A e-marked USB-C cable; a standard cable is limited to 3 A (60 W).
4. Attach an accessory puck to the magnetic dock. Its ID resistor selects the mode and voltage automatically.
5. Use your usual tools (OpenOCD, flashrom, etc.) through the CH347F.

## Status

**Working.** The assembled prototype is tested: the TUSB8041A hub, CH334P and CH347F all enumerate under Windows, and the laptop charges at up to 100 W through the PD output.

## Roadmap

- [x] Finalize schematic and PCB layout
- [x] Order and assemble the first prototype
- [x] Validate hub and CH347F enumeration
- [x] Configure and flash the TPS25751 EEPROM
- [x] Raise laptop charging to 100 W
