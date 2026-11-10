# Changelog

## 0.1.0

The first release: a driver for Texas Instruments' INA226 current, voltage
and power monitor on an I2C bus.

- `Ina226`, the type a board's device table builds with the board's shunt
  resistance, implementing devices-nv's `CurrentMonitor` and `Identifiable`,
  with its own methods for the configuration, the flags, the alert and
  register access.
- The register access as functions over `Ina226Port`, a trait that a type
  holding a bus implements, and the framing as functions over `I2cBus`.
  Each register is its own transaction.
- The manufacturer and die ID check: 0x5449 and a device field of 0x226,
  either revision.  The bring-up resets the part and checks CONFIG.
- CONFIG as four fields (averaging, bus and shunt conversion times, mode),
  the time of a result from them, and a triggered conversion that answers
  its time and does nothing in a continuous mode.
- The calibration from the shunt resistance and the largest current by the
  data sheet's Equations 1 and 2, kept to 15 bits, with the shunt voltage
  limit of 81.92 mV; the current and power converted with the part's real
  step.
- The bus voltage, shunt voltage, current and power in integer units, with
  the math overflow flag.
- The alert as named functions (shunt over and under, bus over and under,
  power over), Conversion Ready, polarity, latch and the limit, with the
  limit written before the function is enabled.
- A device program, `examples/power_read.nv`, and `tests/device_build.sh`,
  which builds it for the nRF52840 DK, the nRF54LM20 DK and the
  FRDM-MCXN947 and runs the emulated board's image under QEMU.
- The suite runs over a double of the part and covers every line of `src/`;
  `tests/coverage.sh` measures it.

### Known

- Hardware verification: NOT VERIFIED.  No board carries an INA226.
- The data sheet read is SBOS547C, June 2011, revised August 2026.  It
  gives a numeric example of the calibration and the three result
  registers, which the suite uses, and none of the alert.
- A reading is six transactions because the data sheet defines no read
  across registers.  A conversion can end between two of them in a
  continuous mode.
- Reading MASK/ENABLE for the overflow flag clears Conversion Ready and a
  latched alert flag; `reading()` returns the flags it read.
- `mon_set_shunt` accepts only the resistance the board was attached with.
- The SMBus alert response, a second channel and an async form are not
  implemented.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, whose
  board packages answer for I2C as absent hardware: on those images
  `begin` answers `false` until the boards drive the buses.  The gap is
  tracked by `features/embedded-devex/nordic-twim-i2c-driver.md`.
- `devices-nv` is the registry range `^0.3.3`, the first release with
  `CurrentMonitor`.
