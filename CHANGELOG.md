# Changelog

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.2) and devices-nv ^0.4.0 (was ^0.3.2), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed; the suite passes without an edit.

The device program builds for the boards its README names, flash nrf52840-dk 17,912 B before and 17,856 B after; nrf54lm20-dk 19,008 B before and 18,932 B after; frdm-mcxn947 19,412 B before and 19,372 B after; nrf52-qemu 8,192 B before and 8,200 B after.

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
- The device program builds for the nRF52840 DK and the nRF54LM20 DK, which
  drive I2C with TWIM0 and TWIM22, so their images carry the reading path:
  22,136 and 23,112 bytes of flash, where before the Nordic buses landed they
  were 12,944 and 13,700 bytes.  A reply from the part over either bus is
  NOT VERIFIED.
