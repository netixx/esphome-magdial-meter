# esphome-magdial-meter

[ESPHome](https://esphome.io) package that reads a meter dial by tracking the **angle** of its rotating
magnet with a 3-axis magnetometer. Built for water meters whose magnet turns once per fixed volume
(e.g. 10 L per turn), works for any meter with a rotating magnet.

Inspired by [tronikos/esphome-magnetometer-water-gas-meter](https://github.com/tronikos/esphome-magnetometer-water-gas-meter),
which counts half rotations on one axis. This package reconstructs the continuous angle of the dial
from two axes instead, so it:

- follows the dial continuously instead of in half-turn (5 L) steps
- knows the rotation direction, so back-and-forth jitter or back-flow is not counted as consumption
- selects the axes and the direction automatically during calibration, and proves a full turn was seen
- makes no assumption on the shape of the signal (a magnet close to the sensor gives sharp peaks)
- follows slow temperature drift of the field (adaptive calibration)

## Installation

Declare the I²C bus in your device config, then include the package once per meter:

```yaml
i2c:
  - id: i2c_bus
    sda: GPIO13
    scl: GPIO16
    frequency: 100kHz

packages:
  water:
    url: https://github.com/netixx/esphome-magdial-meter
    ref: main
    files:
      - path: magdial-meter.yaml
        vars:
          driver: qmc5883p
          volume_per_rotation_initial_value: '10'
    refresh: 1d

preferences:
  flash_write_interval: 5min
```

The counter and calibration are saved to flash at most every `flash_write_interval`. On ESP8266
also set `esp8266: restore_from_flash: true`, otherwise they are kept in RTC memory and lost on
power loss.

Your config must have an `api:` block (the package adds a `set_total` action to it). See
[example-esp32.yaml](example-esp32.yaml) for a full device config.

### Drivers

Set the `driver` var of each meter:

| `driver` | Chip | Address | Notes |
| --- | --- | --- | --- |
| `qmc5883l` | QMC5883L (most GY-271 boards) | 0x0D | native ESPHome |
| `qmc5883p` | QMC5883P (newer GY-271 boards) | 0x2C | external component [mazzhead/esphome-components](https://github.com/mazzhead/esphome-components) (same as tronikos) |
| `hmc5883l` | HMC5883L | 0x1E | native ESPHome |
| `mmc5603` | MMC5603 | 0x30 | native ESPHome |
| `mlx90393` | MLX90393 | 0x0C | native ESPHome. Adafruit boards since 2023 use 0x18: set `i2c_address: '0x18'` |
| `none` | your own | | define the 3 axis sensors yourself, see [drivers/none.yaml](drivers/none.yaml) |

Driver options can be overridden from your config, e.g. for a meter with `id_prefix: cold_`:

```yaml
sensor:
  - id: !extend cold_magnetometer
    oversampling: 4x
```

The package sets no log levels, so it works with any `logger:` level. At the default `DEBUG` level
the driver logs every sample; quiet it with a per-driver level:

```yaml
logger:
  level: DEBUG
  logs:
    qmc5883p: INFO  # or qmc5883l, hmc5883l, mmc5603, mlx90393
```

### Several meters on one device

Include `magdial-meter.yaml` once per meter, each with a unique `id_prefix` and its own
`name_prefix`, bus and driver.

**How many?** Up to **4 meters** per ESP32 comfortably, **8 at most** (one TCA9548A). On an
ESP8266, 2. The limits are I²C wiring and the number of entities, not CPU. Estimated cost per
meter at the default 20 samples per second (estimates, not measured on hardware; 3 meters are
tested in simulation):

| Resource | Per meter | 8 meters | Note |
| --- | --- | --- | --- |
| I²C bus time (100 kHz, ~1 ms per read incl. multiplexer switch) | ~2.5 % | ~20 % | 400 kHz divides it by ~3 |
| Main loop blocked by I²C reads | ~20 ms/s | ~160 ms/s | reads are blocking; still fine for Wi-Fi, Ethernet and the API |
| CPU for the processing | < 1 % | < 5 % | |
| RAM | ~2.5 KB, +2.4 KB while calibrating | ~20 KB, +19 KB if all calibrate at once | an ESP32 has ~200 KB free |
| Flash (code is generated per meter) | ~20 KB | ~160 KB | the app slot is ~1.8 MB |
| Home Assistant entities | ~30, ~20 of them diagnostic | ~240 | set `hide_debug_sensors: 'true'` once set up |

Recommendations:

- **Wiring is the real limit.** Each sensor sits at its meter, often metres away. Several long
  cables on one bus exceed the I²C capacitance budget and cause read errors. A TCA9548A puts each
  cable on its own segment: use one beyond 2 meters, or as soon as cables are long. Keep the bus
  at 100 kHz (or lower for long runs) and use twisted pair (e.g. CAT6) for each sensor cable.
- **Calibrate one meter at a time**: it needs water running through that meter only anyway, and
  it keeps the RAM peak low.
- **Keep `magnetometer_update_interval` at 50 ms** unless a dial turns faster than ~2 turns per
  second; halving it doubles the bus time.
- **More than 8**: chaining multiplexers works, but a second ESP is simpler and isolates faults.

The magnetometers have a **fixed I²C address**, so two identical chips cannot share a bus (ESPHome
does not detect it; the readings would be garbage). Options:

- **Different chips on one bus**: e.g. a QMC5883P (0x2C), a QMC5883L (0x0D) and an HMC5883L (0x1E).
- **One bus per chip**: the ESP32 has 2 hardware I²C controllers (the ESP32-C3 only 1).
- **A TCA9548A multiplexer** (address 0x70): up to 8 identical chips on one bus, one per channel.
  Each channel is a bus id for the meters.

```yaml
i2c:
  - id: i2c_bus
    sda: GPIO13
    scl: GPIO16

tca9548a:
  - address: 0x70
    i2c_id: i2c_bus
    channels:
      - {bus_id: mux_0, channel: 0}
      - {bus_id: mux_1, channel: 1}
      - {bus_id: mux_2, channel: 2}

packages:
  meters:
    url: https://github.com/netixx/esphome-magdial-meter
    ref: main
    files:
      - path: magdial-meter.yaml
        vars: {id_prefix: cold_, name_prefix: 'Cold water ', driver: qmc5883p, i2c_id: mux_0}
      - path: magdial-meter.yaml
        vars: {id_prefix: hot_, name_prefix: 'Hot water ', driver: qmc5883p, i2c_id: mux_1}
      - path: magdial-meter.yaml
        vars: {id_prefix: garden_, name_prefix: 'Garden ', driver: qmc5883p, i2c_id: mux_2,
               volume_per_rotation_initial_value: '1'}
    refresh: 1d
```

Each meter gets its own entities ("Cold water Total", ...), calibration and `set_total` action
(`esphome.<device>_cold_set_total`). See
[example-esp32-three-meters.yaml](example-esp32-three-meters.yaml).

### Vars

Set per meter in `vars:`:

| Var | Default | Meaning |
| --- | --- | --- |
| `id_prefix` | `''` | Prefix of every id and of the `set_total` action. Unique per meter, e.g. `cold_` |
| `name_prefix` | `''` | Prefix of every entity name, e.g. `'Cold water '` |
| `driver` | `qmc5883l` | Magnetometer driver, see [Drivers](#drivers) |
| `i2c_id` | `i2c_bus` | Bus (or TCA9548A channel) of the magnetometer |
| `i2c_address` | `''` | Override the driver's default address |
| `volume_unit` | `L` | Unit of Total, Flow and Volume per rotation |
| `device_class`, `device_icon` | `water`, `mdi:water` | e.g. `gas`, `mdi:meter-gas` |
| `volume_per_rotation_initial_value` | `10` | Volume of one full turn of the magnet. Editable in HA afterwards |
| `magnetometer_update_interval` | `50ms` | Sampling period, see [Sampling rate](#sampling-rate) |
| `calibration_seconds_initial_value` | `180` | Initial calibration time. Editable in HA afterwards |
| `calibration_min_amplitude_initial_value` | `10` | Minimum half-amplitude (µT) of a usable axis. Editable in HA |
| `debug_update_interval` | `2s` | Publish rate of the debug entities to HA |
| `hide_debug_sensors` | `false` | `true` hides the debug entities from HA |
| `flow_update_interval` | `10s` | Flow computation period |
| `flow_deadband_rotations` | `0.01` | Movement (turns) below which Flow reads 0 |
| `adapt_rate` | `0.1` | Weight of each complete turn in adaptive calibration |

Values marked "editable in HA" are restored from flash: changing the var later only affects a
freshly flashed device. Change them in Home Assistant instead.

## Home Assistant entities

| Entity | Category | |
| --- | --- | --- |
| Total | | Volume, `total_increasing`, never decreases. Use it in the Energy dashboard |
| Flow | | Volume per minute |
| Volume per rotation | config | Volume of one turn (10 L) |
| Calibration time | config | Length of the calibration, in s (5 to 3600) |
| Calibration minimal amplitude | config | Weakest usable axis swing, in µT |
| Axis pair | config | Axes used for the angle, set by calibration (x/y, x/z, y/z) |
| Angle mapping | config | **Learned** (default): position within a turn corrected by the learned curve. **Raw**: uncorrected angle |
| Adaptive calibration | config | Follow slow drift of the field after each complete turn |
| Calibrate / Finish calibration now / Cancel calibration | config | Buttons |
| Calibration status | diagnostic | Not calibrated, Calibrating, Ready, Failed: reason |
| Calibration details | diagnostic | Selected axes, turns seen during calibration, clearance |
| Magnetic field X / Y / Z | diagnostic | Raw field, averaged over `debug_update_interval` |
| Magnetic field X / Y / Z min, max | diagnostic | Live extremes while calibrating, calibrated (and adapted) extremes otherwise |
| Dial angle | diagnostic | Position within the current turn, 0 to 360° |
| Rotations total | diagnostic | Turns counted since forever, fractional |
| Learned curve | diagnostic | Raw angle (°) at each tenth of a turn. Evenly spaced (36 72 108 …) means no correction |
| Curve learned turns | diagnostic | Complete turns the curve was learned from (0 to 15, used from 3) |
| Signal clearance | diagnostic | Closest approach of the signal loop to its center over the last turn, in % of the amplitude |

A dashboard card to watch the signal and calibrate (entity IDs for a device named `water-meter`):

```yaml
type: vertical-stack
cards:
  - type: entities
    entities:
      - sensor.water_meter_total
      - sensor.water_meter_flow
      - text_sensor.water_meter_calibration_status
      - text_sensor.water_meter_calibration_details
      - number.water_meter_calibration_time
      - button.water_meter_calibrate
      - button.water_meter_finish_calibration_now
      - sensor.water_meter_dial_angle
      - sensor.water_meter_signal_clearance
  - type: history-graph
    title: Magnetic field
    hours_to_show: 1
    entities:
      - sensor.water_meter_magnetic_field_x
      - sensor.water_meter_magnetic_field_x_min
      - sensor.water_meter_magnetic_field_x_max
      - sensor.water_meter_magnetic_field_y
      - sensor.water_meter_magnetic_field_y_min
      - sensor.water_meter_magnetic_field_y_max
      - sensor.water_meter_magnetic_field_z
      - sensor.water_meter_magnetic_field_z_min
      - sensor.water_meter_magnetic_field_z_max
```

## Calibration

1. Open a tap with a steady flow.
2. Press **Calibrate**. The status shows `Calibrating`.
3. Let the dial make **more than one full turn**. Either wait for **Calibration time** to elapse,
   or watch the physical dial and press **Finish calibration now** once it has turned fully. The
   min/max entities grow while calibrating and stop growing once a full turn is covered.
4. The status shows `Ready` and counting starts. Water used during calibration is not counted:
   use [`set_total`](#setting-the-total) to align Total with the meter.

Choose **Calibration time** from your flow: a 10 L dial at 6 L/min needs 100 s per turn, so the
default 180 s leaves margin. Slower flow or a bigger dial needs a longer time (up to 3600 s).

### How it works

Each turn of the magnet makes the field vector trace a closed loop. Seen through a pair of axes,
each normalised by its min/max, the loop goes once around its center per turn, and the angle
around the center follows the dial. Calibration records the loop and, for each axis pair,
measures:

- the **winding**: how many times the loop went around its center. At least one full turn is
  required, and its sign gives the rotation direction (water flows forward while calibrating);
- the **clearance**: how close the loop comes to its center. Noise near the center would make
  the angle jump, so the pair with the largest clearance wins.

**Why two axes, not one?** One axis alone (as in tronikos' package) can only count threshold
crossings: half a turn (5 L) per step, no direction, and the same value occurs at two angles. Two
axes give the position within the turn and its direction. They must not swing in phase: on a
real meter two axes often mirror each other (in a test log, Y and Z: clearance 1 %), which makes
a flat loop through the center. That pair is useless even if both axes swing strongly, which is
why the pair is chosen by clearance and not by amplitude.

### Angle mapping: Raw or Learned

A magnet close to the sensor gives a non-uniform signal: the raw angle advances faster on some
parts of the turn than others. The count of turns is exact either way, but with **Raw** Total can
lead or lag within a turn by up to ~0.2 turn (2 L on a 10 L dial). It never accumulates.

**Learned** (default) corrects that with a curve of 10 sections of equal volume (1 L each on a
10 L dial), each spanning its own range of raw angle. The curve is learned from complete turns in
normal use, no constant flow needed:

- for each complete turn, the raw angles where 10 %, 20 %, ... of the turn's time had elapsed;
- within one turn a flow change looks like a distortion of the curve, but flow changes happen at
  random positions while the distortion is tied to the dial position. The median of each
  breakpoint over the last 15 turns keeps the distortion and drops the flow changes.

It is used once 3 turns are learned (Raw until then), keeps learning for good, and is reset by a
new calibration. Switching between Raw and Learned takes effect at the next turn boundary, where
both agree, so Total never jumps. In simulation, with random flow changes and stops, the largest
error within a turn (measured from a turn boundary) is 0.05 turn with Learned against 0.42 with
Raw: 0.5 L instead of 4 L on a 10 L dial. Learning runs in Raw mode too, so its entities show
progress either way.

| Status | Fix |
| --- | --- |
| `Failed: no magnetometer data` | Check wiring, I²C address and the log |
| `Failed: low signal...` | Move the sensor closer to the magnet, or lower **Calibration minimal amplitude** |
| `Failed: axes swing in phase...` | Rotate the sensor ~45° on the meter |
| `Failed: incomplete turn (x seen)...` | Run more water or raise **Calibration time** |

A failed or cancelled calibration keeps the previous one. The device log lists, per pair, the
turns seen and the clearance.

**Calibration minimal amplitude** (default 10 µT half-amplitude) only filters out axes that
barely move: raw sensor noise is ~1 µT, and a magnet near the sensor gives hundreds of µT.

**Signal clearance** falling toward 0 means the field changed (sensor moved, big temperature
change): recalibrate. **Adaptive calibration** corrects slow drift by itself, after each complete
turn whose extremes are within 30 % of the calibration.

## Sampling rate

The dial must move less than half a turn between two samples, otherwise turns are miscounted (a
warning is logged above 0.3 turn). Aim for at least 10 samples per turn at your maximum flow:

`magnetometer_update_interval` ≤ `volume_per_rotation` × 6 s / `max_flow (per min)`

A 10 L dial at 60 L/min turns once per 10 s: 50 ms gives 200 samples per turn. A 1 L dial at
60 L/min needs at most 100 ms.

## Setting the total

To make Total match the meter, in Home Assistant go to **Developer tools** >
**Actions**, run `esphome.water_meter_set_total` with `new_total` set to the meter reading.

Changing **Volume per rotation** keeps Total as it is: the new value applies to turns from then on.

## Development

`tests/` holds:

- `test-*.yaml`: a config per driver, validated with `esphome config`;
- `replay-real-log*.yaml`: real readings from a QMC5883P on a 10 L/turn meter
  (`tests/data/real_log_qmc5883p.h`) replayed through the package: calibration must pick x/y
  and count the 1.75 turns of the log, and a calibration over less than a turn must fail;
- `sim-*.yaml`: a simulated magnet orbiting close to the sensor, with noise and thermal drift:
  full run, calibration over a partial turn, long calibration at high flow, and the learned
  angle mapping under random flow with stops.

- `test-multi-*.yaml` and `sim-multi.yaml`: three meters on one device (shared bus, two buses,
  TCA9548A), and three simulated meters calibrating and counting independently.

Each replay and simulation compiles for the host and exits non-zero on failure:

```sh
pip install esphome
esphome config tests/test-*.yaml
for t in replay-real-log:magdial-replay replay-real-log-short:magdial-replay-short \
         sim-host:magdial-sim sim-partial-turn:magdial-sim-partial sim-long-calibration:magdial-sim-long \
         sim-learned-curve:magdial-sim-learned sim-multi:magdial-sim-multi; do
  esphome compile tests/${t%%:*}.yaml && (cd tests && .esphome/build/${t##*:}/.pioenvs/${t##*:}/program) || break
done
```
