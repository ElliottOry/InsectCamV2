# The HM01B0 CSP sensor block

A reusable subcircuit. Everything you need around a Himax HM01B0 in the 16-ball
chip-scale package, and nothing you do not: the sensor, its three supply rails
and their decoupling, the I2C pull-ups, the open-collector inverter that turns
line valid into HSYNC, and the FR4 spacer stack that sets the lens height on the
sensor glass. Power arrives as one rail. Seven wires leave for whatever host you
are using. No regulator, no IMU, no power switch, no radio.

This revision documents the block as built on `himax_integration_5`: a
16.8 x 8.2 mm single-copper-layer board where the block sits beside an
ES2832AA2 radio module (U3), an ICM-42688-P IMU (U1), a TCR3UM33A LDO (VR1) and
a magnetic power switch (U2). Those parts are outside the block. Reference
designators below are the ones on that board.

| | |
|---|---|
| Sensor rail | 3.3 V from VR1 (net named `3.2V`), shared by AVDD and IOVDD |
| Placements | 10 including the sensor |
| Host interface | 7 signals, one supply |
| Escape routing | 1 copper layer, no vias, no ground plane |
| Video mode | 1-bit serial, D0 only |
| Lens height | set by FR4 spacer stack |

---

## Contents

1. [The block](#1-the-block)
2. [Ball assignments](#2-ball-assignments)
3. [Rails and decoupling](#3-rails-and-decoupling)
4. [I2C and MCLK](#4-i2c-and-mclk)
5. [The open collector](#5-the-open-collector)
6. [Land pattern](#6-land-pattern)
7. [Escape routing](#7-escape-routing)
8. [Lens mounting and focusing](#8-lens-mounting-and-focusing)
9. [Block BOM](#9-block-bom)
10. [Host interface](#10-host-interface)
11. [Integration checklist](#11-integration-checklist)
12. [References](#12-references)

---

## 1. The block

A self-contained schematic sheet you can drop into any design. Inside the
boundary: the sensor, its rails and bypassing, the bus pull-ups, and the LVLD
inverter.

```
                       3.3 V in (net 3.2V)
                              |
     +---------+---------+----+-----+----------+----------+
     |         |         |          |          |          |
 [C5 100nF][C4 100nF]    |      [R1 4.7k]  [R2 4.7k]  [R3 4.7k]
 [C8 10uF ][C7 10uF ]    |          |          |          |
     |         |         |          |          |          |
 ....|.........|.........|..........|..........|..........|..........
 :   HM01B1    |         |          |          |          |         :
 :   D1 AVDD --+---------+          |          |          |         :
 :   D4 IOVDD -+  (tied under the array)       |          |         :
 :   D3 DVDD --[C6 100nF]-- GND     |          |          |         :
 :   D2 GND  --- trace to GND ring  |          |          |         :
 :                                  |          |          |         :
 :   A1 SDA  -----------------------+---------------------------->  SDA
 :   B2 SCL  ----------------------------------+------------------>  SCL
 :                                                        |         :
 :   C4 MCLK <---------------------------------------------------  MCLK
 :   C3 PCLKO --------------------------------------------------> PCLKO
 :   B4 D0    --------------------------------------------------> D0
 :   C2 FVLD  --------------------------------------------------> VSYNC
 :                                                        |         :
 :   B1 LVLD ------+   (direct, no series resistor)       |         :
 :                 |                                      |         :
 :             base|  Q2 NPN   collector -----------------+---> HSYNC
 :                 |                                   (inverted)   :
 :                GND  emitter                                      :
 :..................................................................:
          dashed boundary = the block, one supply in
```

Six of the seven signals pass straight through. Frame valid leaves on a net
called **VSYNC**. Line valid goes through Q2 and leaves as **HSYNC**, inverted,
pulled back up to the same rail. One supply in.

![Sensor sheet](img/sensor-sheet.png)

*The sensor sheet as drawn. LVLD reaches the Q2 base with nothing in series; a
collector pull-up makes HSYNC. The no-connect crosses on INT, D1, D2, D3 and TRIG
lock the design into 1-bit mode.*

---

## 2. Ball assignments

Eleven of the sixteen balls are used. The open five are the higher data bits, INT
and TRIG.

```
            col 1        col 2        col 3        col 4

 row A     [ SDA  ]     ( INT  )     ( D3   )     ( D2   )
 row B     [ LVLD ]     [ SCL  ]     ( D1   )     [ D0   ]
 row C     ( TRIG )     [ FVLD ]     [ PCLKO]     [ MCLK ]
 row D     [ AVDD ]     [ GND  ]     [ DVDD ]     [ IOVDD]

 [ ] used in this block      ( ) left open
```

| Ball | Name | Class | Direction | Net on the board | In this block |
|---|---|---|---|---|---|
| A1 | SDA | I2C | bidir | `I2C1/SDA` | Pulled up by R1 to the 3.3 V rail, leaves the block |
| A2 | INT | open | out | | Bring it out if you want motion-detect wake |
| A3 | D3 | open | out | | Needed only for 4-bit or 8-bit mode |
| A4 | D2 | open | out | | Needed only for 4-bit or 8-bit mode |
| B1 | LVLD | video | out | `Net-(HM01B1-LVLD)` | Drives the Q2 base directly, and leaves the block as HSYNC |
| B2 | SCL | I2C | in | `I2C1/SCL` | Pulled up by R2 to the 3.3 V rail, leaves the block |
| B3 | D1 | open | out | | Needed only for 4-bit or 8-bit mode |
| B4 | D0 | video | out | `D0` | The pixel data line in 1-bit mode |
| C1 | TRIG | open | in | | Bring it out if you want hardware single-frame capture |
| C2 | FVLD | video | out | `VSYNC` | Frame valid, straight out at IOVDD level |
| C3 | PCLKO | video | out | `PCLKO` | Pixel clock, straight out. Never buffer this through Q2 |
| C4 | MCLK | video | in | `MCLK` | Master clock from the host |
| D1 | AVDD | power | in | `3.2V` | Analog supply. Takes the rail and the bypass caps directly |
| D2 | GND | gnd | in | `GND` | The only ground ball. Every return current goes through it |
| D3 | DVDD | power | out | `Net-(HM01B1-DVDD)` | Internal 1.5 V LDO output. C6 only, never drive it |
| D4 | IOVDD | power | in | `3.2V` | I/O supply, same rail, tied to D1 under the array. Sets the logic level of every output |

> **There is no reset ball.**
> Software reset at `0x0103` only, which needs working I2C. For a hardware reset,
> cycle the rail.

---

## 3. Rails and decoupling

AVDD and IOVDD share the 3.3 V supply, so there is no sequencing between them.
The rail comes from VR1, a TCR3UM33A 3.3 V LDO; the net is labelled `3.2V` in
the schematic, but it is the regulator's 3.3 V output. DVDD is not an input: it
is the sensor's internal 1.5 V regulator output and takes a bypass capacitor
only. Noise on AVDD shows up as banding in the image, so give its bypass loop
priority.

| Rail | Ball | Nominal | Parts | Placement on the board |
|---|---|---|---|---|
| AVDD | D1 | 3.3 V | C5 100 nF 0201, C8 10 uF 0402 | In a row about 2 mm off the D1 side of the array, fed by a 0.2 mm trace from D1 |
| IOVDD | D4 | 3.3 V | C4 100 nF 0201, C7 10 uF 0402 | In the same row as the AVDD pair. D4 reaches them only through the 0.08 mm tie to D1 under the array (see section 7) |
| DVDD | D3 | 1.5 V, generated internally | C6 100 nF 0201 | About 1.4 mm out, straight off D3 on a 0.2 mm trace. Nothing else touches this net |
| 3.3 V input | | 3.3 V | none dedicated | No block-entry bulk. The nearest bulk is C11 4.7 uF and C10 100 nF beside the radio module, and C12 1 uF at the regulator output |

![Sensor rail ties](img/sensor-rail-ties.png)

*Both sensor rails come off the same supply, each with its own bypass pair.*

### Smaller-footprint alternative

If board space is tight, 1 uF 0402 in place of the two 10 uF parts (C7, C8)
works. The values above are what the board uses and match the source reference
design: 0.1 uF with 10 uF on each of AVDD and IOVDD, 0.1 uF on DVDD.

### IOVDD has no bypass of its own at the ball

On this layout all four rail capacitors sit on the D1 side, so the IOVDD
current loop runs through the thin tie under the array. It works, but it is the
weakest loop on the block. If you re-lay it out, put C4 on the D4 side.

### The single ground ball

D2 is the only ground ball, carrying every return current from all three rails.
This board has no ground plane and no vias: D2 leaves on a 0.2 mm trace and
returns through top-layer copper to the GND ring around the board edge. Keep
that trace short and wide, and keep the bypass capacitor grounds on it.

---

## 4. I2C and MCLK

> **Pull up to IOVDD, not to the host rail.**
> The pull-ups belong inside the block, tied to IOVDD, and 4.7 k is typical. On
> this board R1 and R2 go to the shared 3.3 V rail, which is IOVDD. Pulling them
> to a higher host rail forward-biases the sensor's I/O protection diodes when
> the bus idles high. If the host runs at a different voltage, use a level
> translator outside the block.

The sensor has its own bus, `I2C1`. The IMU sits on a separate bus, `I2C0`, with
its own pull-ups (R4, R5). Neither bus shares pull-ups with the other.

### MCLK before I2C

The sensor will not acknowledge on I2C until MCLK is running, so a bus scan
before that looks like a dead part. Drive C4 between 3 MHz and 36 MHz, and check
the edge at the ball rather than at the host pin.

PCLKO comes from MCLK divided internally by `OSC_CLK_DIV` at `0x3060`, so frame
rate depends on both.

---

## 5. The open collector

Q2 is a small NPN in SOT-883 with its emitter grounded, its base driven straight
from LVLD with no series resistor, and its collector pulled up to the 3.3 V rail
through R3, 4.7 k. That collector node is the block's **HSYNC** output.

| Effect | Consequence |
|---|---|
| Open-collector output | Only ever driven low. Wire-ORable, and the output level is set by the pull-up rail, not the sensor. |
| Inversion | HSYNC is low while LVLD is high. Reference drivers assume active-high line valid, so invert it in firmware. |
| Asymmetric edges | Falling edge is driven, rising edge is RC. Trigger on the falling edge. |
| Level translation, if you want it | Taking the pull-up to a different host rail turns the same circuit into a level shifter, with no other change. |

### Where this circuit belongs

The rising edge is RC, not driven. That is invisible on line valid, which toggles
once per row, and it would wreck a pixel clock. Use this on the sync signals
only; for PCLKO or D0 use a level shifter.

> **The base has no series resistor.**
> Base current is limited only by the sensor pad's own output impedance. A 10 k
> in series is the usual fix if you want it, but it is an option, not a
> correction.

### When to leave the transistor out entirely

Here the transistor is a pure inverting buffer. If you do not need the inversion,
connect B1 straight to a host input and delete Q2 and R3: eight placements. Keep
it for an inverted line valid, a wire-ORable node, or the option to level-shift
later.

---

## 6. Land pattern

The array is not square. Columns are on a 0.50 mm pitch and rows on 0.40 mm,
which changes how you escape it and is easy to get wrong if you assume a uniform
grid.

| Property | Value | Consequence |
|---|---|---|
| Land diameter | 0.26 mm, circular | Non-solder-mask-defined |
| Column pitch | 0.50 mm | Channels between columns are 0.24 mm wide |
| Row pitch | 0.40 mm | Channels between rows are only 0.14 mm wide |
| Array span | 1.50 x 1.20 mm | Ball centre to ball centre, corner to corner |
| Courtyard | body + 1.0 mm | What the KiCad library convention calls for on a WLCSP |

> **The footprint on the board has stray numeric pads.**
> The board uses `camera_flexa:camerah`. Its eleven used balls are named A1 to
> D4, but the five unused ones are numbered `2`, `3`, `4`, `7` and `9` instead of
> A2, A3, A4, B3 and C1. Nothing connects to them, so the board is electrically
> fine, but DRC schematic parity reports "No pad found for pin" on each. The
> `kicad/Himax_HM01B0.kicad_mod` in this package names all sixteen correctly;
> swap to it to clear the warnings.

---

## 7. Escape routing

Everything is on the top copper layer with no vias. Trace widths step down
toward the array: 0.2 mm for power and ground, 0.13 mm for the slower signals
and the outer escapes, 0.08 mm for anything that passes between balls. The
board's minimum clearance is set to 0.02 mm, so this layout needs a process that
holds 0.08 mm lines and about 0.03 mm spaces.

The outer balls escape straight out. Row D (AVDD, GND, DVDD, IOVDD) goes off
the row-D edge, MCLK and D0 off the column-4 edge, SDA and LVLD off the
column-1 corner. The inner four are B2, B3, C2 and C3; three are used (SCL,
FVLD, PCLKO). The two channel widths are not equivalent:

- **Between columns: 0.24 mm.** A 0.08 mm trace leaves 0.08 mm each side.
- **Between rows: 0.14 mm.** The same trace leaves 0.03 mm each side.

The board uses both:

| Net | Ball | Width | Route |
|---|---|---|---|
| SCL | B2 | 0.13 mm | Through the column 1/2 channel, out past row A |
| FVLD (`VSYNC`) | C2 | 0.08 mm | Into the row B/C channel, out past column 4 |
| PCLKO | C3 | 0.08 mm | Into the row C/D channel, out past column 4 |
| 3.3 V tie | D1 to D4 | 0.08 mm | Along the row C/D channel, linking AVDD and IOVDD under the array |
| 3.3 V feed | D1 | 0.08 mm | Column 1/2 channel, row B/C channel, then the column 2/3 channel out past row A |

> **The row channels work here, but only just.**
> Three of the routes above sit in a 0.14 mm row channel with about 0.03 mm to
> the neighbouring lands. If your fab cannot hold that, escape the inner balls
> through the column channels only, or drop a via.

> **There is no ground plane.**
> With one copper layer and nothing on the back, ground is a top-layer trace
> network tied to a copper ring around the board outline. A two-layer re-spin
> should add a plane under the sensor and a via at D2.

### Placement order around the array

1. **Ground at D2.** A short, wide trace off the row-D edge. Everything else
   references it.
2. **C6 on DVDD.** The closest part on the block, returning to the same ground.
3. **C5 and C4, the 100 nF pair.** On this board both sit on the D1 side; put C4
   by D4 if you have room.
4. **C8 and C7, the 10 uF bulk.** These can sit a few millimetres out.
5. **R1 and R2, then Q2 and R3.** Pull-ups anywhere on the I2C run; the
   transistor group near the block edge on the host side.

---

## 8. Lens mounting and focusing

The CSP arrives as a bare package with a glass surface and no lens holder, so the
optics are entirely your problem. The method below sets lens height with a stack
of laser-cut FR4 shims, found experimentally against a live image rather than
calculated. It assumes the sensor is already running, the lens is in hand, and
the FR4 cutout files exist. The cutouts are sized to sit on the HM01B0 glass
without obstructing the sensor ROI.

Using both 100 um and 150 um FR4 lets the height be adjusted in 50 um increments
across the useful range. The sequence is: find the height experimentally, bond a
stack at that height, verify the stack, glue the stack to the sensor, then align
and glue the lens, checking the live feed at every stage.

### Materials

| Item | Why |
|---|---|
| HM01B0 with live feed | Focus is judged on the image, not measured. The feed has to be running for most of the procedure |
| Lens | The spacer height found here is specific to this lens |
| FR4 spacers, 100 um and 150 um | Cut from the provided files. Two thicknesses give 50 um resolution in the stack |
| Plastic tweezers | Metal risks scratching the glass and the lens |
| Small clamp | Holds the FR4 stack tight while the glue wicks in |
| Thin cyanoacrylate (regular super glue) | Bonds the FR4 layers, and fixes the lens |
| BSI Maxi-Cure | Bonds the FR4 stack to the sensor glass |
| Digital micrometer | To measure the finished stack. Nominal FR4 thickness is not accurate enough here |

### Procedure

**1. Find the correct spacer height.**
Run the live feed while testing the lens. Start with a single 100 um spacer:
place it on the sensor, then place the lens on top with plastic tweezers, move
the lens into approximate alignment, and check the image for focus.

If it is not in focus, increase the spacer height and repeat. With 100 um and
150 um pieces you can walk up through 100, 150, 200, 250, 300 um and so on until
you find the height that produces the best focus.

Nothing is glued in this step. The only purpose is to determine the required
lens-to-sensor spacing.

**2. Glue the FR4 stack together.**
With the height known, build a permanent stack from the required pieces. Stack
them dry and clamp them tightly. While clamped, apply a small amount of regular
thin super glue to the *edge* of the stack and let capillary action pull it
between the layers. Maxi-Cure is too thick to wick in.

After the glue cures, remove the clamp and measure the finished stack with a
digital micrometer. Measure at several points to confirm the stack stayed flat
and that the final thickness is close to the value that focused well during
testing.

**3. Verify the finished stack before gluing it down.**
Place the completed stack on the sensor with no adhesive and set the lens on top.
Check the live feed again to confirm the image is still in focus.

This catches any height change introduced while bonding the pieces. If focus is
no longer correct, adjust the stack before anything is permanently attached to
the sensor.

**4. Attach the stack to the HM01B0.**
Apply a small amount of Maxi-Cure to the mounting surface and place the stack
onto the HM01B0 glass. Align the cutout so the sensor ROI stays completely
unobstructed. Use only enough glue to secure the stack, and keep adhesive out of
the sensor area and the optical path. Let it cure before continuing.

**5. Recheck focus and align the lens.**
With the stack permanently mounted, place the lens on top without glue and watch
the live feed. The stack now fixes the height, so the lens only needs lateral
movement. Use plastic tweezers to shift it until the image is centred and focus
looks correct. Verify focus again before attaching anything.

**6. Glue the lens in place.**
Remove the lens, apply a small amount of super glue to the top of the FR4
platform, and set the lens back down. Keep the live feed running while
positioning it: make small adjustments with plastic tweezers until the image is
centred and focused, then hold it until the glue is stable.

After the adhesive fully cures, check the image one final time for focus,
centring, and any vignetting or obstruction.

> **Never put wet glue between the FR4 layers.**
> Applying adhesive between the pieces before clamping adds an uncontrolled glue
> thickness and changes the final lens height, which is the one dimension this
> whole procedure exists to control. Clamp dry, then wick the glue in from the
> edge.

### Working notes

- Keep the live feed running whenever possible while testing or positioning the lens.
- Use plastic tweezers around both the lens and the sensor.
- Measure the finished stack rather than trusting nominal FR4 thickness.
- Recheck focus after every permanent gluing step.
- Once a spacer thickness is known for a particular lens, record it. It becomes
  the starting point for every future build with that lens, and step 1 collapses
  to a single confirmation.

---

## 9. Block BOM

| Ref | Value | Package | Purpose |
|---|---|---|---|
| HM01B1 | HM01B0 | 16-ball CSP (`camera_flexa:camerah`) | Himax always-on QVGA monochrome image sensor |
| C5 | 100 nF | 0201 | AVDD high-frequency bypass |
| C8 | 10 uF | 0402 | AVDD local bulk |
| C4 | 100 nF | 0201 | IOVDD high-frequency bypass |
| C7 | 10 uF | 0402 | IOVDD local bulk, feeds the output drivers |
| C6 | 100 nF | 0201 | DVDD bypass at ball D3. The only thing allowed on this net |
| R1 | 4.7 k | 0201 | SDA pull-up to the 3.3 V rail (IOVDD) |
| R2 | 4.7 k | 0201 | SCL pull-up to the 3.3 V rail (IOVDD) |
| R3 | 4.7 k | 0201 | Q2 collector pull-up to the 3.3 V rail. Makes the HSYNC output |
| Q2 | NPN, small signal | SOT-883 | Open-collector inverter on LVLD. Its collector is the HSYNC output |

Delete Q2 and R3 if you want line valid non-inverted and do not need an
open-collector output. That takes the block to eight placements.

Not in the block, but on the same board: C1, C2, C3 (IMU and module
decoupling), C9 to C12 (regulator and module supply), R4 and R5 (`I2C0`
pull-ups for the IMU), U1, U2, U3, VR1 and test pads J1 to J6.

---

## 10. Host interface

The host on `himax_integration_5` is the ES2832AA2 module (U3). The pin column
shows where each block signal lands on it.

| Signal | Dir | Level | Rate | U3 pin | Host responsibility |
|---|---|---|---|---|---|
| 3.3 V | in | 3.3 V | DC | 8, VCC | Clean analog-grade supply, shared by all three rails. Cycling it is the only hardware reset available |
| GND | in | | DC | 1, 13, 18, 26 | Board has no plane; ground is the top-layer ring and traces |
| MCLK | in | IOVDD | 3 to 36 MHz | 22, P0.18 | Must be running before any I2C transaction is attempted |
| SCL | in | IOVDD, open drain | <= 400 kHz | 4, P0.00 | Pulled up inside the block. Do not add a second pull-up. P0.00 is also XL1, so no 32 kHz crystal on XL1/XL2 |
| SDA | bidir | IOVDD, open drain | <= 400 kHz | 3, P0.05 | Sensor answers at 0x24 |
| PCLKO | out | IOVDD | pixel rate | 21, P0.07 | Sample D0 on this edge. Do not buffer or level-shift with a transistor |
| D0 | out | IOVDD | pixel rate | 25, P0.20 | One bit per PCLKO edge, eight edges per pixel |
| FVLD (`VSYNC`) | out | IOVDD | frame rate | 20, P0.08 | Active high, non-inverted. Frames the burst |
| HSYNC | out | IOVDD, open collector | line rate | 2, P0.06 | Inverted line valid, active low. Invert in firmware or set the capture peripheral polarity accordingly |

Nothing in the block latches, buffers or retimes the pixel path. PCLKO, D0 and
FVLD arrive at the host exactly as the sensor produced them, so all the capture
timing work lives on the host side.

---

## 11. Integration checklist

- [ ] Bulk capacitance is near the block. (This board relies on C11 4.7 uF by the module rather than a dedicated block-entry part.)
- [ ] AVDD and IOVDD are tied to the same supply, so there is no sequencing question between them.
- [ ] DVDD has a capacitor and nothing else. No trace leaves that net.
- [ ] D2 has a short, wide path to ground: a via to a plane on two layers, or a 0.2 mm trace to the ground ring on one.
- [ ] I2C pull-ups reference IOVDD, and there is no second pair on the host side.
- [ ] The Q2 collector is pulled up to the sensor rail, and LVLD reaches the base with nothing in series.
- [ ] Q2 is on LVLD only. PCLKO and D0 go straight through.
- [ ] Firmware treats HSYNC as inverted line valid, or the capture peripheral polarity is set for active low.
- [ ] Inner-ball escapes fit the fab: 0.08 mm traces in the 0.24 mm column channels, or in the 0.14 mm row channels only if about 0.03 mm space is achievable.
- [ ] Footprint pad numbers are the ball designators A1 to D4, with no stray numeric pads. (Fails on the current board footprint; see section 6.)
- [ ] Host code starts MCLK before the first I2C scan.
- [ ] Spacer cutout leaves the sensor ROI completely unobstructed.
- [ ] Finished FR4 stack has been measured with a micrometer, not assumed from nominal thickness.
- [ ] Focus rechecked on the live feed after every permanent gluing step.

---

## 12. References

- `himax_integration_5.kicad_sch` and `himax_integration_5.kicad_pcb`: the
  schematic and board this document is taken from.
- `kicad/Himax_HM01B0.kicad_mod` and `kicad/himax_hm01b0.kicad_sym` in this
  package: footprint and symbol with all sixteen balls named A1 to D4.
- Himax HM01B0 datasheet: ball map, register map (`0x0103` software reset,
  `0x3060` `OSC_CLK_DIV`) and the source decoupling values.
