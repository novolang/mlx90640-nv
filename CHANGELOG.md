# Changelog

## 0.1.1 — 2026-11-12

The release depends on devices-nv 0.4.0 and embedded-hal-nv 0.4.0.

- `Mlx90640` implements devices-nv's `ThermalImager`, with the effect list
  `[hw, mutate]`: `thermal_cols` (32), `thermal_rows` (24), `thermal_start` (the time an
  image takes, two subpage periods, and -1 when the sensor does not answer),
  `thermal_read` and `thermal_rate_mhz` (half the subpage rate).  `thermal_read` reads both
  subpages and calculates the image at an emissivity of 0.95 (`EMISSIVITY_MILLI`) with its
  working buffer in its own frame; on `false` the buffer is empty.
- `Mlx90640.frame(emissivity_milli, var w, var out)` stays as the richer form, with another
  emissivity and the caller's read buffer.
- `MlxFrame` is now an alias of devices-nv's `ThermalFrame`, the same `Vec[i32; 768]`.
- New module functions `thermal_capture`, `image_us` and `image_rate_mhz`.

## 0.1.0

The first release: a driver for Melexis's MLX90640 32 by 24 infrared array sensor.

- `Mlx90640`, the type a program builds from the board's I2C bus, timer and the sensor's
  wiring (`wiring_i2c(0, 0x33)`). `begin` reads the 832-word EEPROM at 400 kHz, in pieces of
  16 words, into module state. `frame` reads both subpages into a caller buffer of 834 words
  and calculates the 768 pixels into a caller buffer of `i32` millidegrees Celsius.
- The calculation follows section 11 of the datasheet: the supply voltage and the ambient
  temperature, the gain, the offset with the row and column terms, Kta and Kv, the compensation
  pixel and the temperature gradient coefficient, the emissivity and reflected temperature, the
  sensitivity, a first estimate, and the four temperature ranges by the corner temperatures. The
  chess pattern, the interleaved pattern and a calibration in either are handled. The EEPROM is
  decoded with integers and the compensation is done in `Float`.
- The ambient temperature as `Thermometer`, the device ID words as `Identifiable`, the refresh
  rate from 0.5 to 64 Hz, the resolution, the pattern, the pixel defects and their correction.
- `frame_read` waits at most twice the subpage period, clears the data-ready bit with the
  reference driver's value, and answers a read that met an update as a data error.
- The device program `examples/thermal_reader.nv`, an ambient and hottest-pixel reader. Built
  for the nRF52840 DK it is 24,732 B of flash and for the nRF54LM20 DK 26,180 B.
- Line coverage of `src/` is 100 %, measured with `tests/coverage.sh`.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the sensor. The cost of a
  frame on a board and the stack the program needs are not measured.
- The datasheet's worked example (section 11.2.2.6.2 and 11.2.2.7) subtracts a compensation
  pixel of 21.63 from a pixel of subpage 0, where the pixel of subpage 0 takes 25.67, and the 21.63
  disagrees with the formula that gives it (25.78). Its final temperature, 80.363, is therefore
  not reproduced; the first estimate for its printed V_IR is (80.3633), and the whole pixel is
  checked against an independent calculation. The same example prints the pixel's EEPROM
  address as 0x258F where its row and column give 0x25AF.
- The datasheet prints its revision date as October 21, 2026.
- The calibration mode bit (bit 11 of the word 0x240A) is the reference driver's; the datasheet
  does not describe the word.
- The calculation of a pixel is about 300 floating-point operations in binary64; a frame on a
  Cortex-M4 or M33 is soft-float for binary64. A binary32 version is not written.
- At a refresh of 32 Hz the bus has to run at 1 MHz, and at 64 Hz the reads in pieces of 32
  bytes do not fit in a subpage period.
- The EEPROM is module state, so a program has one MLX90640. A trait method takes a plain `self`
  and cannot keep it.
- `devices-nv` 0.3.3 has no trait for a thermal array; the package defines its own frame API.
- Not implemented: writing the EEPROM, the step mode and subpage-repeat modes, the I2C
  configuration register.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module also checks at
  `rt` and `system`.
