# Changelog

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.3), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed; the suite passes without an edit.

The device program builds for the boards its README names, flash nrf52840-dk 17,480 B before and 17,424 B after; nrf54lm20-dk 18,576 B before and 18,516 B after; frdm-mcxn947 19,152 B before and 19,084 B after; nrf52-qemu 9,056 B before and 9,064 B after.

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
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, which
  drive I2C with TWIM0 and TWIM22, so their images carry the reading path:
  21,624 and 22,584 bytes of flash, where before the Nordic buses landed
  they were 14,076 and 14,840 bytes.  A reply from the part over either bus
  is NOT VERIFIED.
- `devices-nv` is the registry range `^0.3.3`, the first release with
  `CurrentMonitor`.
