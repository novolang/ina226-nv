# ina226-nv

A driver for the INA226, a current, voltage and power monitor from Texas
Instruments with an I2C interface and an alert pin, specified in the data
sheet "INA226 36V, 16-Bit, Ultra-Precise I2C Output Current, Voltage, and
Power Monitor With Alert", document SBOS547C, June 2011, revised August
2026.  The driver implements the `CurrentMonitor` and `Identifiable` traits
of [devices-nv](https://novo-lang.org/packages/devices-nv): the bus
voltage, the shunt voltage, the current and the power of one rail, in
integer units, and the manufacturer and die ID check.

**Hardware: NOT VERIFIED.**  No board carries an INA226 yet.  The device
program below builds for the nRF52840 DK and the nRF54LM20 DK.  The
driver's functions run over a double of the part that follows the data
sheet's register map, its Equations 3 and 4 and its bus protocol.  A
reading from a part on a board has not been taken.

## Install

```sh
novo pkg add ina226-nv
```

The device program is `examples/power_read.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
novo build --target=frdm-mcxn947 src/main.nv
```

## The device program

`examples/power_read.nv` reads a breakout over I2C once a second and prints
the bus voltage in millivolts, the current in microamps and the power in
microwatts over RTT:

```
ina226 mV=3302 uA=41250 uW=136207
```

The breakout's SDA and SCL go to the board's I2C pins and its A0 and A1
pins to ground, which puts the part at 0x40.  The load goes through the
breakout's shunt, a 0.1 ohm resistor.  The program brings the part up,
tells it the shunt and the largest current it expects, and reads.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_current.{ CurrentMonitor }
use ina226.{ Ina226 }

// The part's address with A1 and A0 at ground.
const ADDR = 0x40

@tier(embedded)
fn main() [hw]
    // The board's shunt is 0.1 ohm: 100000 micro-ohms.
    let part = Ina226.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR), 100000)
    if not part.begin()
        hal.uart.write("ina226: no answer\n")
        return
    // 800 mA at most: 80 mV across the shunt.
    if not part.mon_set_shunt(0, 100000, 800000)
        hal.uart.write("ina226: shunt refused\n")
        return
    loop
        match part.mon_read(0)
            Some(r) =>
                hal.uart.write("ina226 mV=")
                hal.uart.write(str.from_int(r.bus.mv))
                hal.uart.write(" uA=")
                hal.uart.write(str.from_int(r.current.ua))
                hal.uart.write(" uW=")
                hal.uart.write(str.from_int(r.power.uw))
                hal.uart.write("\n")
            None => hal.uart.write("ina226: no result\n")
        hal.timer.delay_ms(1000)
```

`begin` reads the manufacturer ID and the die ID, resets the part, and
checks that CONFIG holds its power-on value.  It answers `false` for a part
that does not answer, for one that is not an INA226, and for one that does
not reset.  `mon_set_shunt` programs the Calibration register.  The full
program, which also waits the time `mon_start` answers, is
`examples/power_read.nv`.  A script builds it for the boards below by
making it the `main` of a scratch package that depends on this one.  It
also runs the emulated board's image under QEMU, where no part answers, and
checks that the program reports it.

```sh
bash tests/device_build.sh
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 14,076 B | 2,022 B | 2,660 B |
| `nrf54lm20-dk` | 14,840 B | 1,980 B | 2,644 B |
| `frdm-mcxn947` | 23,628 B | 7,898 B | 2,748 B |

The numbers are from the build's memory report.  Flash also holds the
runtime, the compiler's support library and the program's tables.  The
arena, 65,536 B on the Nordic boards, is the runtime's heap and is reserved
whether or not the program uses it.

The two Nordic board packages answer for I2C as absent hardware, so the
build removes the code that follows the first bus read, and their rows are
the image without the driver's reading path.  The novo-lang
repository tracks the missing Nordic I2C driver as a feature.  The FRDM-MCXN947 board
package drives I2C, and its row is the image with the driver's code in it.
On the Nordic boards the program prints `ina226: no answer`.

### The effects

Every function of the driver that reaches the bus or the ALERT pin declares
`[hw]`, and nothing else.  The calibration, the register conversions and
the tables are pure.  The driver performs no input or output of its own and
builds no list of its own.  The bus functions of devices-nv build one short
frame per transaction inside a `system` block and release it when the call
returns.  The package checks at the embedded tier, so a program of any tier
may call it.  The program above declares `[hw]` for the same reason.

## What it is

A current monitor sits on a supply rail.  A shunt resistor of a fraction of
an ohm is in series with the load, and the monitor measures the small
voltage across it, which Ohm's law turns into a current.  It also measures
the voltage at its VBUS pin and multiplies the two into a power.

The INA226 does not know the shunt's resistance.  The program tells it by
writing the Calibration register, and the part then computes its Current
and Power registers.  Until then they read 0, which is the same as a load
that draws nothing.  The driver therefore answers `None` to `mon_read`
until a shunt has been set, and a bus voltage, which needs no calibration,
has its own call.

| Register | Address | Step | What it holds |
|---|---|---|---|
| CONFIG | 0x00 | | reset, averaging, conversion times, mode; power-on value 0x4127 |
| SHUNT | 0x01 | 2.5 µV | signed; full scale 81.92 mV |
| BUS | 0x02 | 1.25 mV | bit 15 is always 0; full scale 40.96 V |
| POWER | 0x03 | 25 current steps | unsigned |
| CURRENT | 0x04 | one current step | signed |
| CALIBRATION | 0x05 | | 15 bits |
| MASK/ENABLE | 0x06 | | the alert function and its flags |
| ALERT LIMIT | 0x07 | | compared with the register the function names |
| MANUFACTURER ID | 0xFE | | 0x5449 |
| DIE ID | 0xFF | | 0x2260 or 0x2261: device 0x226, revision 0 or 1 |

The calibration is the data sheet's Equations 1 and 2.  With Current_LSB
the current step and R the shunt, CAL = 0.00512 / (Current_LSB · R), and
the Current_LSB is at least the largest current expected over 2^15.  The
data sheet's example is 15 A at most across 2 mΩ: a step of 1 mA gives CAL
= 2560.  The part then computes the current as the shunt register times CAL
over 2048 and the power as the current times the bus register over 20000.
A shunt of 8000 steps (20 mV) is 10000 steps of current, 10 A, and a bus
voltage of 9584 steps (11.98 V) makes 4792 steps of power, 119.82 W.

The driver picks the smallest whole-microamp step that covers the largest
current and keeps CAL at 32767 or less, computes CAL by truncation, and
converts the Current register with the part's real step,
5 120 000 000 / (CAL · R) microamps, so a truncated CAL leaves no gain
error.  It refuses a largest current that would drop more than 81.92 mV
across the shunt.

CONFIG holds four fields.

| Field | Bits | Values |
|---|---|---|
| AVG | 11:9 | 1, 4, 16, 64, 128, 256, 512 or 1024 averages |
| VBUSCT | 8:6 | 140, 204, 332, 588, 1100, 2116, 4156 or 8244 µs |
| VSHCT | 5:3 | the same times |
| MODE | 2:0 | 0 and 4 power-down; 1 shunt, 2 bus, 3 both, triggered; 5, 6 and 7 the same, continuous |

A result takes the averages times the conversion times of the signals the
mode measures: 2.2 ms at the power-on values.  A write to CONFIG halts a
conversion in progress and starts one from the new settings.

The alert pin goes active when one function the program picks compares
true: the shunt voltage over or under the limit, the bus voltage over or
under it, or the power over it.  The Conversion Ready flag can also assert
the pin.  The pin is open-drain, active low unless the polarity bit is
set, and it latches until MASK/ENABLE is read if the latch bit is set.

## What the package contains

| Module | What is in it |
|---|---|
| `ina226` | The types `Ina226`, `Ina226Config`, `Ina226Cal`, `Ina226Flags` and `Ina226Reading`, the trait `Ina226Port`, the bus framing, the registers, the identity, the configuration, the calibration, the readings, the alert and the bring-up |

## How to choose an entry point

A program on a board uses `Ina226`, built by `attach` with the board's bus,
its GPIO, the wiring and the shunt resistance in micro-ohms.  The resistance
is a fact about the board, as the address is, and the driver keeps it.  The
wiring's `irq` is the ALERT pin.  The type implements `CurrentMonitor` and
`Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w, shunt_micro_ohm)` | the part at `w.addr`; nothing is sent |
| `begin()` | makes the bus ready, checks the IDs, resets, checks CONFIG |
| `mon_set_shunt(0, shunt, max_ua)` | `CurrentMonitor`: programs the calibration; refuses a shunt other than the board's, a largest current that drops more than 81.92 mV, and a channel other than 0 |
| `mon_read(0)` | the bus voltage, shunt voltage, current and power, or `None` before a shunt was set |
| `mon_bus_voltage(0)` | the bus voltage alone, one transaction |
| `mon_start()` | 0 in a continuous mode; otherwise a triggered conversion and its time in microseconds; −1 when the part did not answer |
| `device_id()` | `Identifiable`: the die ID's device field, 0x226, with the revision as `rev` |
| `reading()` | the measurement with the MASK/ENABLE flags it was read with |
| `flags()`, `ready()` | the flags; the Conversion Ready flag. Both clear it |
| `config()`, `set_config(c)`, `set_averaging(n)`, `set_conversion_times(b, s)`, `set_mode(n)` | the CONFIG fields; each write is read back |
| `set_alert(func, ready, active_high, latch, limit)` | the alert function, with the limit in the register's steps |
| `alert_pin()` | the level of the ALERT pin, −1 for a wiring with none |
| `calibration()`, `reg(r)` | the Calibration register; any register |

The alert functions are named: `ALERT_NONE`, `ALERT_SHUNT_OVER`,
`ALERT_SHUNT_UNDER`, `ALERT_BUS_OVER`, `ALERT_BUS_UNDER` and
`ALERT_POWER_OVER`.  `shunt_limit_raw(uv)`, `bus_limit_raw(mv)` and
`power_limit_raw(uw, cal, shunt)` turn a limit in microvolts, millivolts
and microwatts into the register's steps.

A program that holds its own bus, or a test with a double of the part,
implements `Ina226Port` over its bus and calls the module's functions,
which take the port: `ina226.reading_with_flags(port, shunt)`,
`ina226.cal_write(port, shunt, max_ua)`.  The framing is `i2c_read_into`
and `i2c_write`, generic over `I2cBus`.  The conversions are
`shunt_uv_of`, `bus_mv_of`, `current_ua_of` and `power_uw_of`, which need
no bus.

The package gives no async form.  `async fn` is refused at the embedded
tier the package checks at; the time to wait is what `mon_start` answers.

## The rules a user needs

1. Call `begin`, then `mon_set_shunt`, before the first `mon_read`.  `begin`
   resets the part, and a reset clears the calibration.
2. `mon_set_shunt` takes the resistance the board was attached with.  A
   different one would make every current and power wrong, and is refused.
3. A reading is six transactions, one register each: the sheet defines a
   read as the pointer write and one register, and promises no read that
   runs on into the next.  In a continuous mode a conversion can end
   between two of them; wait `mon_start`'s time in a triggered mode for a
   reading from one conversion.
4. Reading MASK/ENABLE clears the Conversion Ready flag and a latched Alert
   Function Flag.  A reading reads it last, for the math overflow flag, and
   so clears them.  A program that latches alerts reads `reading()`, which
   returns the flags it read, or reads `flags()` itself.
5. With the math overflow flag set the current and power registers are not
   measurements.  `mon_read` answers the bus and shunt voltages with
   `overflow` set and the current and power 0.
6. A current is positive from the supply side of the shunt to the load
   side.  The Power register is unsigned and reads the magnitude.
7. The shunt voltage saturates at 81.92 mV.  The driver refuses a largest
   current past it; a load that exceeds the largest current saturates the
   registers.
8. The ALERT pin needs a pull-up on the board.

## What is not included

- The SMBus alert response and the high-speed mode of the bus.
- More than one channel: the part measures one rail.  The INA3221 measures
  three.
- An async form, see above.
- The INA219, an older part with another register map; `i2c-devices-nv`
  drives it.
- An averaged wait: a program that wants to wait for a result polls
  `ready()` or waits `mon_start`'s time.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits
  this driver implements, the wiring record and the register helpers it
  uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` and `GpioOut` traits and the board handles.
- i2c-devices-nv drives the INA219 beside other sensors.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus at 0x40, with the power-on values, the read-only registers, a
Current register and a Power register computed from the others by the data
sheet's equations, a MASK/ENABLE register whose flags clear when it is read,
and a read that answers one register and nothing longer.  It checks the
framing of every entry point, the configuration, the calibration, the
readings, the alert, the identity and the bring-up with its refusals.  The
suite covers every line of `src/`; `tests/coverage.sh` measures it.

The calibration and the registers are checked against the data sheet's
worked example (Section 6.5.1 and Table 6-1): a 2 mΩ shunt, a step of 1 mA,
CAL = 2560, a shunt register of 8000, a bus register of 9584, a current of
10000 and a power of 4792, which the driver reads as 10 A and 119.8 W.  The
data sheet gives no numeric example of the alert; the limit values in the
suite are computed from the register steps.

**Hardware: NOT VERIFIED.**  See the top of this page.

The sources read for this driver, for their fixes and not ported, are the
data sheet above and Zephyr's `drivers/sensor/ti/ina2xx`.  Embassy and
esp-hal hold no INA226 driver.  Each fix kept is named in `src/ina226.nv` where it is kept:

- Zephyr computes the calibration from the devicetree and does not check
  that it fits 15 bits or is not 0.  This driver raises the step until it
  fits and refuses a shunt for which none does.
- Zephyr rounds CAL to the closest; this driver truncates, and converts the
  current with the real step.
- Zephyr does not reset the part, and does not refuse a shunt that
  saturates the shunt register.
- The data sheet's Table 7-1 gives the die ID as 0x2260 or 0x2261; the
  driver accepts either.
- devices-nv's reference double lets a read run on into the next register,
  which the data sheet does not promise.

## Licence

Apache-2.0.
