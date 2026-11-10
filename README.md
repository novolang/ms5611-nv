# ms5611-nv

A driver for the MS5611-01BA03, a barometric pressure sensor from TE
Connectivity (formerly Measurement Specialties) with an I2C and an SPI
interface, specified in the data sheet "MS5611-01BA03 Barometric Pressure
Sensor, with stainless steel cap", document DA5611-01BA03_011, 06/2017.
The driver implements the `Barometer` and `Thermometer` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv).  It reads the
calibration PROM, checks its CRC-4, and compensates the readings with the
data sheet's first and second order formulas in integers.

**Hardware: NOT VERIFIED.**  No board carries an MS5611 yet.  The device
program below builds for the nRF52840 DK and the nRF54LM20 DK.  The
driver's functions run over a double of the part that follows the data
sheet's commands and bus protocols.  A reading from a part on a board has
not been taken.

## The device program

`examples/baro_read.nv` reads a breakout over I2C once a second and prints
the temperature in millidegrees and the pressure in pascals over RTT:

```
ms5611 mC=20070 Pa=100009
```

The breakout's SDA and SCL go to the board's I2C pins, its PS pin to the
supply, which selects I2C, and its CSB pin to the supply, which puts the
part at 0x76.  CSB to ground puts it at 0x77.  The program brings the part
up, converts the pressure and then the temperature at the finest
oversampling, and waits the time each takes.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio, BoardTimer }
use dev_bus
use ms5611.{ Ms5611 }

// The part's address with CSB high.
const ADDR = 0x76

@tier(embedded)
fn main() [hw]
    var part = Ms5611.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    let timer = BoardTimer { id: 0 }
    if not part.begin(timer)
        hal.uart.write("ms5611: no answer\n")
        return
    loop
        match part.sample(timer)
            Some(s) =>
                hal.uart.write("ms5611 mC=")
                hal.uart.write(str.from_int(s.milli_c))
                hal.uart.write(" Pa=")
                hal.uart.write(str.from_int(s.pa))
                hal.uart.write("\n")
            None => hal.uart.write("ms5611: no result\n")
        hal.timer.delay_ms(1000)
```

`begin` resets the part, waits the 2.8 ms the PROM takes to reload (3 ms
on the timer), reads the eight PROM words and checks their CRC-4.  It
answers `false` for a part that does not answer, for a PROM that reads all
zeros or all ones, and for a PROM whose CRC does not match.  `sample`
starts the pressure conversion at the oversampling code in use, waits its
time on the timer, reads the result, does the same for the temperature,
and compensates.

The program is built for the boards below by a script that makes it the
`main` of a scratch package depending on this one.  It also runs the
emulated board's image under QEMU, where no part answers, and checks that
the program reports it.

```sh
bash tests/device_build.sh
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 12,944 B | 1,282 B | 2,660 B |
| `nrf54lm20-dk` | 13,700 B | 1,240 B | 2,644 B |
| `frdm-mcxn947` | 24,308 B | 8,678 B | 2,748 B |

The numbers are from the build's memory report.  Flash also holds the
runtime, the compiler's support library and the program's tables.  The
arena, 65,536 B on the Nordic boards, is the runtime's heap and is reserved
whether or not the program uses it.  The compensation is 64-bit integer
arithmetic.

Neither Nordic board package drives I2C yet.  `BoardI2c` is absent
hardware there, every transfer is refused, and the program prints
`ms5611: no answer`.  The build removes the code that follows the first
bus read, so the two Nordic rows are the image without the driver's reading
path.  The feature `features/embedded-devex/nordic-twim-i2c-driver.md` in
the novo-lang repository tracks the missing driver.  The FRDM-MCXN947 board
package drives I2C, and its row is the image with the driver's code in it.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The compensation, the CRC and the conversion-time functions
are pure.  The driver performs no input or output of its own and builds
no list of its own.  The bus functions of devices-nv build one short frame
per transaction inside a `system` block and release it when the call
returns.  The package checks at the embedded tier, so a program of any
tier may call it.  The program above declares `[hw]` for the same reason.

## What it is

A barometer measures air pressure.  Air pressure at sea level is about
101 325 pascals (Pa), and falls by about 12 Pa for each metre of height
near the ground.  The package gives no altitude: that needs the pressure
at sea level where the board is, which changes with the weather.

The MS5611 has a piezo-resistive sensor and a 24-bit converter.  It has
five commands and no registers to write:

| Command | Byte | What it does |
|---|---|---|
| Reset | 0x1E | the PROM reloads in 2.8 ms |
| Convert D1 | 0x40 + 2k | the pressure at OSR 256 · 2^k, k = 0 to 4 |
| Convert D2 | 0x50 + 2k | the temperature at OSR 256 · 2^k |
| Read ADC | 0x00 | the 24-bit result, MSB first |
| Read PROM | 0xA0 + 2n | the 16-bit word n, 0 to 7, MSB first |

The PROM holds eight words: factory data, the six coefficients C1 to C6,
and the serial code with a 4-bit CRC.  Each part has its own.

| Oversampling code | OSR | Wait before the ADC read |
|---|---|---|
| 0 | 256 | 600 µs |
| 1 | 512 | 1,170 µs |
| 2 | 1024 | 2,280 µs |
| 3 | 2048 | 4,540 µs |
| 4 | 4096 | 9,040 µs |

The waits are the maximum column of the data sheet's conversion-time row,
and `ms5611.conversion_us(code)` answers them.  A higher OSR gives less
noise and takes longer.  The data sheet lists the RMS noise at OSR 4096 as
0.012 mbar and 0.002 °C.

The compensation is the data sheet's Figure 2 and Figure 3, written as
integer operations with arithmetic shifts.  It needs both conversions.

| Quantity | Formula | Unit |
|---|---|---|
| dT | D2 − C5 · 2^8 | |
| TEMP | 2000 + dT · C6 / 2^23 | 0.01 °C |
| OFF | C2 · 2^16 + C4 · dT / 2^7 | |
| SENS | C1 · 2^15 + C3 · dT / 2^8 | |
| P | (D1 · SENS / 2^21 − OFF) / 2^15 | 0.01 mbar = 1 Pa |

Below 20 °C the second order terms apply: T2 = dT² / 2^31, OFF2 =
5 · (TEMP − 2000)² / 2 and SENS2 = 5 · (TEMP − 2000)² / 4, and below
−15 °C OFF2 gains 7 · (TEMP + 1500)² and SENS2 gains
11 · (TEMP + 1500)² / 2.  Then TEMP, OFF and SENS lose T2, OFF2 and SENS2.
The reading is in millidegrees Celsius and pascals.

The part has two bus interfaces, chosen by its PS pin.

| | I2C | SPI |
|---|---|---|
| Address | 0x76 with CSB high, 0x77 with CSB low | chip select low |
| A command | one-byte write | the byte in one chip-select window |
| A read | the command, a repeated start, the data | the command, then the data clocked out |
| Clock | 400 kHz asked for | 4 MHz, mode 3 (mode 0 or 3, up to 20 MHz) |

Unlike the BMP390's SPI read, the answer to an MS5611 SPI read starts at
the first byte after the command; no dummy byte is dropped.

## What the package contains

| Module | What is in it |
|---|---|
| `ms5611` | The types `Ms5611` (I2C) and `Ms5611Spi` (SPI), `Ms5611Prom`, `Ms5611Sample`, `Ms5611Calc` and `Ms5611State`, the trait `Ms5611Port`, the bus framing, the commands, the PROM and its CRC, the compensation and the bring-up |

## How to choose an entry point

A program on a board uses `Ms5611` for a part on I2C and `Ms5611Spi` for a
part on SPI.  Each is built by `attach` with the board's bus, its GPIO and
the wiring, as a board's device table builds it.  Both implement
`Barometer` and `Thermometer`.  The device value holds the PROM, the
oversampling code and the last temperature result, and a program keeps it
in a `var`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin(timer)` | resets, waits `RESET_US`, reads and checks the PROM |
| `reset()`, `load_prom()` | the same in two steps, for a program that waits itself |
| `set_osr(code)`, `osr()` | the oversampling code, 0 to 4; 4 after `attach` |
| `sample(timer)` | one reading, blocking on the timer: `Ms5611Sample` with `milli_c` and `pa` |
| `temp_start()`, `temp_read()` | `Thermometer`: start the D2 conversion and answer its time in microseconds; the temperature in millidegrees with the second order term |
| `take_temperature()` | read the D2 result and keep it for `baro_read` |
| `baro_start()`, `baro_read()` | `Barometer`: start the D1 conversion; the pressure in pascals, with the kept temperature |
| `prom()` | the `Ms5611Prom` the driver holds |

A program that cannot block on a timer runs the conversions itself.  The
sequence for a pressure is:

1. `temp_start()`, wait the microseconds it answers, `take_temperature()`.
2. `baro_start()`, wait the microseconds it answers, `baro_read()`.

The temperature changes slowly, so a program may take it once for many
pressures.  `baro_read` answers `None` until a temperature has been taken.
Both starts answer −1 when the part did not acknowledge, and a read before
the wait ends answers `None` rather than a wrong value.

A program that holds its own bus, or a test with a double of the part,
implements `Ms5611Port` over its bus and calls the module's functions,
which take the port: `ms5611.start(port, timer, var state)`,
`ms5611.measure(port, timer, state)`.  The framing of each bus is
`i2c_command`, `i2c_read_into`, `spi_command` and `spi_read_into`, generic
over `I2cBus` and `SpiBus`.  The compensation is `first_order`,
`compensate`, `temperature_centi` and `prom_crc4`, which need no bus.

The package gives no async form.  `async fn` is refused at the embedded
tier the package checks at, and the waits are public.  A program on a
scheduler awaits `TimerAsync.sleep_us(ms5611.conversion_us(code))` between
`start_pressure` and `adc_read`.

## The rules a user needs

1. Wait the conversion time before `adc_read`.  The part answers 0 for a
   read before the conversion ends, the conversion goes on, and the next
   result is wrong (data sheet, Conversion sequence).  The driver treats 0
   as no result, and so does it for a second read of one result.
2. Do not start a conversion while one runs.  The result of both is wrong.
3. Wait `RESET_US` after a reset before reading the PROM.
4. Read the PROM once.  `begin` keeps it; `sample` and the trait reads use
   the kept copy, so a pressure is one command and one read.
5. A device value changes only through the `var self` methods: `begin`,
   `load_prom`, `set_osr` and `take_temperature`.  A trait method takes a
   plain `self` and does not change the temperature it was given.
6. The part has no identification register.  The PROM's CRC and the
   all-zeros and all-ones refusals are the check that a part is there.

## What is not included

- `Identifiable`: the part has no identification register.
- An async form, see above.
- The MS5607, whose coefficients and second order terms differ.  Its
  constants are 2^17, 2^16, 2^6 and 2^7 in the first order terms and
  61 / 16, 2, 15 and 8 in the second; this package does not accept them.
- An altitude, which needs the pressure at sea level (see above).

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits
  this driver implements, the wiring record and the register helpers it
  uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus`, `SpiBus`, `GpioOut` and `Timer` traits and the board
  handles.
- bmp390-nv drives Bosch's BMP390 barometer with the same bus conventions.

## Tests

`novo test` runs the module over a double of the part that answers on an
I2C bus and on an SPI bus, with the conversions whose ADC read answers 0
before they end and after the result was read once, the PROM that stops
answering part way, and the SPI answer without a dummy byte.  It checks
the framing of every command, the PROM read and its CRC, the compensation,
the settings, the bring-up and its refusals, and the readings through the
traits.  The suite covers every line of `src/`; `tests/coverage.sh`
measures it.

The compensation is checked against the data sheet's worked example (Figure
2: C1 to C6 = 40127, 36924, 23317, 23282, 33464, 28312; D1 = 9085466; D2 =
8569150), which gives dT = 2366, TEMP = 2007, OFF = 2420281617, SENS =
1315097036 and P = 100009.  The example is at 20.07 °C and reaches no
second order term.  The branches below 20 °C and below −15 °C are checked
against vectors computed by a separate transcription of Figure 2 and
Figure 3 in Python integers with arithmetic shifts.  The data sheet gives
no numeric example of them.

The CRC-4 is the algorithm of application note AN520, which the data sheet
names and which was not fetched: it is read from Linux's
`drivers/iio/pressure/ms5611_core.c`, which cites it.  The suite checks it
against a separate transcription of that code.  The data sheet's Figure 16
puts the CRC in bits 3 to 0 of word 7; AN520's code clears the whole low
byte of word 7 before the sum, and so does this driver.  No PROM read from
a real part has been checked, because none has been read.

**Hardware: NOT VERIFIED.**  See the top of this page.

The sources read for this driver, for their fixes and not ported, are the
data sheet above, Linux's `drivers/iio/pressure/ms5611_core.c` and
Zephyr's `drivers/sensor/meas/ms5607`.  Embassy and esp-hal hold no
barometer driver (`planning/PERIPHERAL-DRIVER-CANDIDATES.md`, section 3, in
the novo-lang repository).  Each fix kept is named in `src/ms5611.nv` where
it is kept:

- Zephyr divides where Linux shifts, which differ by one count where an
  intermediate is negative, as every reading below 20 °C has.  The driver
  shifts.
- Zephyr's driver is the MS5607's, and its constants are not the MS5611's;
  a driver for the MS5611 that takes them reads a wrong pressure that is
  plausible.  The MS5611's are used.
- Neither driver refuses an ADC result of 0, or a PROM of all zeros, whose
  CRC matches an all-zero CRC field.  This one does.
- Linux waits 3 to 4 ms after a reset where the data sheet says 2.8 ms.
  The driver answers 3 ms.

## Licence

Apache-2.0.
