# mlx90640-nv

A driver for Melexis's MLX90640, an infrared array sensor of 32 by 24 pixels with an
I2C interface. The sensor is calibrated at the factory and holds its calibration in
an EEPROM; the host reads the raw counts of half the array at a time, a subpage, and
calculates the object temperature of each pixel. The package reads the EEPROM once,
reads a frame as the two subpages into buffers the caller owns, and calculates the
768 temperatures in millidegrees Celsius by the steps of the datasheet "MLX90640 32x24
IR array", 3901090640, revision 13. It implements `Thermometer` for the ambient
temperature and `Identifiable` of [devices-nv](https://novo-lang.org/packages/devices-nv).

**Hardware verification: NOT VERIFIED.** No board on the bench carries this sensor, so
the driver has run only against a double of the sensor, in the host suite, and the
calculation against the datasheet's worked example and an independent calculation of
the datasheet's steps. The device program below builds for the nRF52840 DK and the
nRF54LM20 DK and has not been flashed. Its stack use is NOT VERIFIED.

## Install

```sh
novo pkg add mlx90640-nv
```

The device program is `examples/thermal_reader.nv`.  Make it the `main` of a
package that depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/thermal_reader.nv` reads the EEPROM, then reads and calculates a frame about
every second period of the sensor's default 2 Hz refresh, and prints a line over the
board's console:

```
ambient 25 C  hottest 31 C at row 12 col 16
```

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardTimer }
use dev_thermo.{ TempMilliC, Thermometer }
use dev_bus
use mlx90640.{ Mlx90640, MlxWords, MlxFrame }

// Writes `n`, 0 or more, in decimal.  Formatting into a string would allocate,
// which @tier(embedded) refuses, so each digit goes out as a byte.  An Int has
// at most 19 decimal digits, which bounds both loops.
@tier(embedded)
fn digits(n: Int) [hw]
    var div = 1
    for _k in 0..18
        if div <= n / 10
            div = div * 10
    for _k in 0..19
        if div > 0
            hal.uart.write_byte(48 + n / div % 10)
            div = div / 10

// Writes a temperature in whole degrees, rounded toward zero, with a minus sign.
@tier(embedded)
fn degrees(milli_c: Int) [hw]
    var m = milli_c
    if m < 0
        hal.uart.write("-")
        m = -m
    digits(m / 1000)

// The index of the hottest pixel of the frame.
@tier(embedded)
fn hottest(var frame: MlxFrame) -> Int
    var best = 0
    for i in 1..768
        if frame[i] > frame[best]
            best = i
    best

@tier(embedded)
fn main() [hw, mutate]
    bsp.board.init()
    let sensor = Mlx90640.attach(BoardI2c { id: 0 }, BoardTimer { id: 0 }, dev_bus.wiring_i2c(0, 0x33))
    // The part is ready 80 ms after its supply (datasheet section 10.5).
    hal.timer.delay_ms(80)
    if not sensor.begin()
        hal.uart.write("no answer from the sensor at 0x33\n")
    var raw: MlxWords = Vec.new()
    var frame: MlxFrame = Vec.new()
    while true
        // Emissivity 0.95, the value of Adafruit's port.
        if sensor.frame(950, var raw, var frame)
            match sensor.temp_read()
                Some(t) =>
                    hal.uart.write("ambient ")
                    degrees(t.milli_c)
                None => hal.uart.write("ambient ?")
            let h = hottest(var frame)
            hal.uart.write(" C  hottest ")
            degrees(frame[h] as Int)
            hal.uart.write(" C at row ")
            digits(h / 32)
            hal.uart.write(" col ")
            digits(h % 32)
            hal.uart.write("\n")
        else
            hal.uart.write("no answer\n")
            hal.timer.delay_ms(500)
```

The program is `examples/thermal_reader.nv`. A board package that depends on this
package builds it as its `main`. `tests/device_build.sh` does so from a scratch package
for the two boards below and for the emulated board, where it runs the image and checks
that the program reports that the sensor does not answer:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program |
|---|---|---|
| `nrf52840-dk` | 24,732 B | 13,330 B |
| `nrf54lm20-dk` | 26,180 B | 13,746 B |

The numbers are from the build's memory report. Flash also holds the runtime and the
compiler's support library, which includes the soft floating-point routines the calculation
uses. The arena, the runtime's heap, is reserved whether or not the program uses it; the
driver allocates nothing in it except the list the bus answers each read of 16 words with,
released when the read returns.

Wiring: the breakout's SDA and SCL to the board's first I2C bus, 3.3 V to its supply
(the datasheet asks for 3.3 V within 50 mV). The sensor answers at 0x33. The Nordic DKs
drive I2C with a TWIM, on the pins `docs/hal/README.md` names under "The Nordic DKs' buses".
A reply from the sensor over a DK's bus is NOT VERIFIED until the driver rig carries one.

## What the package contains

| Module | Holds |
|---|---|
| `mlx90640` | The `Mlx90640` type, the `MlxPort` trait the register functions run over, the decoding of the EEPROM, the calculation of the supply voltage, the ambient temperature and each pixel, the defective-pixel correction, the control register functions, and the subpage read. |

## The frame

A subpage is read into an `MlxWords`, a `Vec[u16; 834]`: words 0 to 767 are the pixels, 768
to 831 the auxiliary area (Vbe, the compensation pixels, the gain, PTAT and the supply
reading), 832 the control register and 833 the subpage number. `Mlx90640.frame` reads both
subpages, calculates the pixels of each and fills an `MlxFrame`, a `Vec[i32; 768]` of
millidegrees Celsius, row by row from the top left. The reflected temperature is taken as the
ambient less 8 degrees, the open-air value of the datasheet (section 11.2.2.9).

The package keeps the EEPROM, 832 words, in module state, so a program has one sensor.
`begin` reads it at 400 kHz, the speed the datasheet allows for the EEPROM.

## The calculation

The calibration is decoded with integer arithmetic. The compensation uses `Float`, which is
binary64: it multiplies quantities from about 1e-9 (a pixel's sensitivity) to about 1e10 (an
absolute temperature to the fourth power) and takes three fourth roots a pixel, and a
fixed-point value of 32 or 64 bits covers neither the range nor the precision of the
datasheet's example. The roots are a Newton iteration written in the package, since the
embedded tier does not admit `math.sqrt`. A pixel takes about 300 floating-point operations;
the cost of a frame on a board has not been measured.

The steps are the datasheet's (section 11.2.2): the supply voltage, the ambient temperature,
the gain, the offset, supply and temperature compensation of each pixel, the compensation
pixel's share (the temperature gradient coefficient TGC), the emissivity, the sensitivity,
a first estimate of the object temperature, and the estimate again with the slope of its
temperature range. The four ranges are below 0 degrees, 0 degrees to CT3, CT3 to CT4 and above
CT4; the corner temperatures and slopes are in the EEPROM.

## The rules a user needs

1. Both subpages are needed for an image. The sensor alternates them, subpage 0 first, one
   each refresh period, so a frame takes two periods.
2. The refresh rate is the rate of a subpage, from 0.5 to 64 Hz (codes 0 to 7). A frame read
   takes 1,668 bytes; at 400 kHz that is about 43 ms, so a refresh of 16 Hz or more needs a
   faster bus, up to 1 MHz (`bus_hz`), and 64 Hz does not fit in pieces of 32 bytes.
3. `refresh_set`, `resolution_set` and `chess_set` change control register 1 and read it back.
4. The factory calibration is for the chess pattern (control bit 12 set) and 18-bit resolution.
   Another pattern or resolution is corrected for as the datasheet describes (sections 11.1.3.1
   and 11.2.2.1).
5. `frame_read` waits at most twice a subpage period for the new-data bit. A read that meets
   the sensor's own update leaves 0x7FFF in a word, and the read answers `ERR_DATA`.
6. A broken pixel (its EEPROM word is 0) is set to `BROKEN_MILLI_C`. `defects_correct`
   replaces broken pixels and outliers (EEPROM bit 0 set) by the mean of their good neighbours.
7. The datasheet's first frames after power-up are not settled: the thermal behaviour takes
   up to 4 minutes (section 12.2.2).
8. The ambient temperature (`temp_read`) reads the auxiliary area and the control register.
9. `Identifiable` answers the three ID words 0x2407 to 0x2409 as one number in `who`.

## What is not included

- The temperature range of the object beyond -40 to 300 degrees; the calculation does not clamp.
- The 1 MHz bus mode of a board that cannot drive it: `bus_hz` opens the bus at the speed
  asked and answers whether the wiring has an address.
- Writing the EEPROM, the step mode and the subpage-repeat modes of control register 1, and
  the I2C configuration register 0x800F.
- A calculation without floating point.

## Related packages

- [amg8833-nv](https://novo-lang.org/packages/amg8833-nv) reads a thermal array of 8 by 8
  pixels.
- [devices-nv](https://novo-lang.org/packages/devices-nv) defines `Thermometer` and
  `Identifiable`; [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) defines
  `I2cBus` and `Timer`.

## Tests

`novo test --isolate tests/mlx90640_tests.nv` runs the suite on the host.
It decodes the calibration of the datasheet's Table 12 and checks the
supply voltage (3.3186 V), the ambient temperature (39.184 degrees), the gain, the offset
(-75), the sensitivity (1.26223e-7), the compensation pixel and the first estimate for the
example's V_IR (80.3633 degrees) against section 11.2.2. The full pixel is checked against an
independent calculation of the datasheet's steps for the four temperature ranges, the
interleaved pattern, emissivity and the 19-bit resolution. The datasheet's example takes the
compensation pixel of subpage 1 for a pixel of subpage 0 and prints 21.63 for it where its
formula gives 25.78, so its final 80.363 is not reproduced; the test comments say how. The
frame functions are checked against a double of the sensor, with a failure injected at every
transaction. `bash tests/coverage.sh` measures the line coverage of `src/`.

## Sources

- MLX90640 32x24 IR array, Datasheet, 3901090640, revision 13 (Melexis).
- Read for fixes, not ported: Melexis's mlx90640-library (`MLX90640_API.c`, Apache-2.0) and
  its history, and Adafruit_MLX90640.

## Licence

Apache-2.0.
