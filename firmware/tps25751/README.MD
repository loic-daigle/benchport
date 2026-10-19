# TPS25751 configuration (USB PD controller)

The TPS25751 that charges the laptop and handles the data-role swap loads its firmware and configuration from an external EEPROM (**U9, CAT24C512**). **This EEPROM must be flashed before the board will work.**

With a blank EEPROM the TPS25751 stays idle. You get no PD contract, no power to the laptop and no data-role swap, so the hub never connects and **the computer won't detect Benchport at all.**

## Files

| File | What it is |
|---|---|
| `config_TPS25751_F8091166_fullFlash.bin` | Ready-to-flash Full Flash image (TI firmware F809.11.66 + Benchport configuration). This is what goes into the EEPROM. |
| `benchport_tps25751_config.json` | Project file for TI's TPS25751 Application Customization Tool. Open it to change the configuration and export a new `.bin`. |

## Configuration in this image

| Setting | Value |
|---|---|
| Type-C state machine | Source only (the laptop can never feed power back into the dock) |
| USB data role | Initiates and accepts swap to UFP; rejects swap to DFP |
| Power-role swaps | Swap to sink rejected |
| USB | USB Communication Capable, USB3 Gen1 |
| Source PDO 1 | 5 V / 3 A (PP5V) |
| Source PDO 2 | 20 V / 5 A (PPHV), up to 100 W |
| Type-C current before PD | 3 A |
| GPIO1 | `ufp_dfp_event` (#29), push-pull → hub USB_VBUS detect |
| GPIO2 | `cable_orientation_event` (#3), push-pull → upstream SuperSpeed mux select |

The 20 V / 5 A PDO needs a **5 A e-marked USB-C cable** to the laptop. With a standard cable, the laptop is limited to 3 A (60 W).

## Hardware context

- **Boot mode:** ADCIN1 = ADCIN2 = GND, which selects SafeMode and boots from the EEPROM. The TPS25751's own I2C target address on I2Ct is **0x21**.
- **EEPROM:** CAT24C512 (64 kB) on the I2Cc bus at **0x50** (A0–A2 = GND, WP = GND).
- **Programming access:** the I2Ct test pads (SCL / SDA / GND), with 10 kΩ pull-ups on board.

## How to flash

### Option 1: through the TPS25751 (recommended)

The TPS25751 programs its own EEPROM. You only talk to it over the I2Ct pads.

1. Power the board from the USB-C PD input. Leave the laptop port unplugged.
2. Connect a 3.3 V USB-to-I2C adapter (for example an FT232H, a CH341A in I2C mode, a Bus Pirate, or TI's USB2ANY) to the **I2Ct** pads: SCL, SDA, GND.
3. Check the connection: read the Mode register (0x03) at address 0x21. A blank EEPROM reads `PTCH`, meaning the device is waiting for a patch.
4. Write the image with the TPS25751 4CC command sequence described in the Technical Reference Manual (SLVUCR8) and TI app note SLVAFL1:
   - `FLad`: set the start address
   - `FLwd`: write the data, in 32-byte chunks
   - `FLvy`: verify
   - `GAID`: reset, so the device reloads from the EEPROM

   (4CC commands go to the CMD1 register, 0x08, with their data in DATA1, 0x09.)
5. After the reset, the Mode register should read `APP ` instead of `PTCH`.

### Option 2: write the EEPROM directly

Write `config_TPS25751_F8091166_fullFlash.bin` from address 0x0000 to the CAT24C512 at **0x50** on the I2Cc bus, using any I2C EEPROM programmer.

The TPS25751 is also a controller on that bus and reads the EEPROM at power-up, so make sure it isn't driving the bus while you program. Option 1 avoids this issue entirely.

## Check that it worked

1. Power-cycle the board.
2. Connect the laptop to the upstream USB-C port. It should start charging.
3. A USB-C PD tester inline on the laptop cable should show two source PDOs: **5 V / 3 A** and **20 V / 5 A**.
4. The computer should detect the hubs (TUSB8041A, CH334P) and the CH347F.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Mode register still reads `PTCH` | EEPROM is blank, or the write failed verification. Flash again. |
| Laptop charges but no USB devices appear | Data-role swap not happening. Check GPIO1 (should be high once the laptop is the host). |
| USB 3 works in only one plug orientation | GPIO2 (cable orientation) isn't reaching the upstream mux select. |
| Laptop charges at 60 W instead of 100 W | The cable isn't 5 A e-marked, or the input charger can't supply 20 V / 5 A. |

## Changing the configuration

1. Open `benchport_tps25751_config.json` in **TI's TPS25751 Application Customization Tool**.
2. Edit the settings (for example the PDOs, to match a different charger).
3. Export a **Full Flash** binary and flash it as described above.

Keep **Source only** and the **UFP swap** settings. The dock relies on them to work as a dock.

## License note

The `.bin` file contains TI firmware, which is covered by TI's own license terms, not this repository's license. The configuration (`.json`) and this documentation are part of Benchport.
