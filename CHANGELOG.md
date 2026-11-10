# Changelog

## 0.1.0

The first release: a driver for TE Connectivity's MS5611-01BA03 barometric
pressure sensor on an I2C or an SPI bus.

- `Ms5611` (I2C) and `Ms5611Spi` (SPI), the types a board's device table
  builds, implementing devices-nv's `Barometer` and `Thermometer`, with
  their own methods for the bring-up, the oversampling, a blocking reading
  and the temperature the barometer needs.
- The commands as functions over `Ms5611Port`, a trait that a type holding
  a bus implements, and the framing of each bus as functions over
  `I2cBus` and `SpiBus`.  An SPI read has no dummy byte.
- The reset, the PROM read of eight words and the CRC-4 check of
  application note AN520, which also refuses a PROM of all zeros or all
  ones.
- The pressure and temperature conversions at the five oversampling ratios
  with the data sheet's maximum conversion times as `conversion_us`, and
  `RESET_US`.  An ADC result of 0 is no result.
- The first order compensation of the data sheet's Figure 2 and the
  second order compensation of Figure 3, in 64-bit integers with
  arithmetic shifts: the temperature in millidegrees and the pressure in
  pascals.
- A device program, `examples/baro_read.nv`, and `tests/device_build.sh`,
  which builds it for the nRF52840 DK and the nRF54LM20 DK and runs the
  emulated board's image under QEMU.
- The suite runs over a double of the part on both buses and covers every
  line of `src/`; `tests/coverage.sh` measures it.

### Known

- Hardware verification: NOT VERIFIED.  No board carries an MS5611.
- The data sheet's worked example is at 20.07 °C and reaches no second
  order term.  The suite's vectors for the branches below 20 °C and below
  -15 °C come from a separate transcription of the data sheet's formulas.
- The data sheet gives the CRC-4 by reference to AN520, which was not
  fetched.  The algorithm is read from Linux's `ms5611_core.c` and clears
  the low byte of PROM word 7 before the sum, where the data sheet's
  Figure 16 puts the CRC in the low four bits.
- The divisions of the compensation are arithmetic shifts (round toward
  minus infinity), as in Linux; Zephyr's MS5607 driver divides (round
  toward zero).  The two differ by one count below 20 °C.
- No async form: `async fn` is refused at the embedded tier.
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, whose
  board packages answer for I2C and SPI as absent hardware: on those
  images `begin` answers `false` until the boards drive the buses.  The
  gap is tracked by `features/embedded-devex/nordic-twim-i2c-driver.md`.
