# esphome-magdial-meter

[ESPHome](https://esphome.io) package that reads a meter dial by tracking the **angle** of its rotating
magnet with a 3-axis magnetometer. Built for water meters whose magnet turns once per fixed volume
(e.g. 10 L per turn), works for any meter with a rotating magnet.

Inspired by [tronikos/esphome-magnetometer-water-gas-meter](https://github.com/tronikos/esphome-magnetometer-water-gas-meter),
which counts half rotations on one axis. This package reconstructs the continuous angle of the dial
from two axes instead, so it:

- resolves fractions of a turn (1 L steps and better on a 10 L dial, where half-turn counting gives 5 L steps)
- knows the rotation direction, so back-and-forth jitter or back-flow is not counted as consumption
- selects the axes and the direction automatically during calibration
- follows slow temperature drift of the field (adaptive calibration)

## Installation

```yaml
substitutions:
  volume_unit: 'L'
  volume_per_rotation_initial_value: '10'
  i2c_sda: GPIO13
  i2c_scl: GPIO16

packages:
  meter:
    url: https://github.com/netixx/esphome-magdial-meter
    ref: main
    files: [magdial-meter.yaml, drivers/qmc5883p.yaml]
    refresh: 1d

preferences:
  flash_write_interval: 5min
```

Pick exactly one driver:

| Driver file | Chip | Notes |
| --- | --- | --- |
| `drivers/qmc5883l.yaml` | QMC5883L | native ESPHome |
| `drivers/qmc5883p.yaml` | QMC5883P | external component [mazzhead/esphome-components](https://github.com/mazzhead/esphome-components) (same as tronikos) |
| `drivers/hmc5883l.yaml` | HMC5883L | native ESPHome |
| `drivers/mmc5603.yaml` | MMC5603 | native ESPHome |
| `drivers/mlx90393.yaml` | MLX90393 | native ESPHome. Adafruit boards since 2023 use address 0x18, see the file |

Driver options can be overridden from your config, e.g. a wider range if the field saturates:

```yaml
sensor:
  - id: !extend magnetometer
    range: 800µT
```

Your config must have an `api:` block (the package adds the `set_total` action to it). See
[example-esp32.yaml](example-esp32.yaml) for a full device config.

The package sets no log levels, so it works with any `logger:` level. At the default `DEBUG` level
the driver logs every sample; quiet it with a per-driver level:

```yaml
logger:
  level: DEBUG
  logs:
    qmc5883p: INFO  # or qmc5883l, hmc5883l, mmc5603, mlx90393
```

### Substitutions

| Substitution | Default | Meaning |
| --- | --- | --- |
| `volume_unit` | `L` | Unit of Total, Flow and Volume per rotation |
| `volume_per_rotation_initial_value` | `10` | Volume of one full turn of the magnet. Editable in HA afterwards |
| `i2c_sda`, `i2c_scl`, `i2c_frequency` | `GPIO4`, `GPIO5`, `50kHz` | I²C bus. Lower the frequency for long cables |
| `magnetometer_update_interval` | `50ms` | Sampling period, see [Sampling rate](#sampling-rate) |
| `calibration_seconds_initial_value` | `180` | Initial calibration time. Editable in HA afterwards |
| `calibration_min_amplitude_initial_value` | `3` | Minimum half-amplitude (µT) of a usable axis. Editable in HA |
| `debug_update_interval` | `2s` | Publish rate of the debug entities to HA |
| `hide_debug_sensors` | `false` | `true` hides the debug entities from HA |
| `flow_update_interval` | `10s` | Flow computation period |
| `flow_deadband_rotations` | `0.01` | Movement (turns) below which Flow reads 0 |
| `adapt_rate` | `0.1` | Weight of each complete turn in adaptive calibration |

Values marked "editable in HA" are restored from flash: changing the substitution later only
affects a freshly flashed device. Change them in Home Assistant instead.

## Home Assistant entities

| Entity | Category | |
| --- | --- | --- |
| Total | | Volume, `total_increasing`, never decreases. Use it in the Energy dashboard |
| Flow | | Volume per minute |
| Volume per rotation | config | Volume of one turn (10 L) |
| Calibration time | config | Length of the calibration, in s (5 to 3600) |
| Calibration minimal amplitude | config | Weakest usable axis swing, in µT |
| Axis pair | config | Axes used for the angle, set by calibration (x/y, x/z, y/z) |
| Rotation direction | config | Set by calibration. Flip it if Total doesn't increase while water runs |
| Adaptive calibration | config | Follow slow drift of the field after each complete turn |
| Calibrate / Finish calibration now / Cancel calibration | config | Buttons |
| Calibration status | diagnostic | Not calibrated, Calibrating, Detecting direction, Ready, Failed: reason |
| Calibration details | diagnostic | Selected axes, phase between them, amplitudes |
| Magnetic field X / Y / Z | diagnostic | Raw field, averaged over `debug_update_interval` |
| Magnetic field X / Y / Z min, max | diagnostic | Live extremes while calibrating, calibrated (and adapted) extremes otherwise |
| Dial angle | diagnostic | Position within the current turn, 0 to 360° |
| Rotations total | diagnostic | Turns counted since forever, fractional |
| Signal quality | diagnostic | ~100 % when the calibration fits the signal |

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
      - sensor.water_meter_signal_quality
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
3. Let the dial make **at least one full turn**. Either wait for **Calibration time** to elapse, or
   watch the physical dial and press **Finish calibration now** once it has turned fully. The
   min/max entities grow while calibrating and stop growing once a full turn is covered.
4. Keep the water running: the status shows `Detecting direction` for about a quarter turn,
   then `Ready`.
5. Water used during step 3 is not counted. Use [`set_total`](#setting-the-total) to align Total with the meter.

Choose **Calibration time** from your flow: a 10 L dial at 6 L/min needs 100 s per turn, so the
default 180 s leaves margin. Slower flow or a bigger dial needs a longer time (up to 3600 s).

Calibration records the min/max of every axis, and where the other axes were at each extreme. A
rotating magnet makes each axis a sinusoid of the dial angle with its own offset, amplitude and
phase; from those it picks the pair of axes with the largest usable signal (weaker amplitude ×
sin(phase between them)). Two axes swinging in phase carry no direction information, so the pair
with the strongest *out-of-phase* signal wins, not simply the two strongest axes.

| Status | Fix |
| --- | --- |
| `Failed: no magnetometer data` | Check wiring, I²C address and the log |
| `Failed: low signal...` | Move the sensor closer to the magnet, or lower **Calibration minimal amplitude** |
| `Failed: axes swing in phase...` | Rotate the sensor ~45° on the meter |
| `Failed: incomplete turn...` | Run more water or raise **Calibration time** |

A failed calibration keeps the previous one. **Cancel calibration** also restores the previous one.

**Signal quality** drifting well away from 100 % means the field changed (sensor moved, big
temperature change): recalibrate. **Adaptive calibration** corrects slow drift by itself, after
each complete turn whose extremes are within 30 % of the calibration.

## Sampling rate

The dial must move less than half a turn between two samples, otherwise turns are miscounted (a
warning is logged above 0.3 turn). Aim for at least 10 samples per turn at your maximum flow:

`magnetometer_update_interval` ≤ `volume_per_rotation` × 6 s / `max_flow (per min)`

A 10 L dial at 60 L/min turns once per 10 s: 50 ms gives 200 samples per turn. A 1 L dial at
60 L/min needs at most 100 ms.

## Setting the total

To make Total match the meter, in Home Assistant go to **Developer tools** >
**Actions**, run `esphome.water_meter_set_total` with `new_total` set to the meter reading.

After changing **Volume per rotation**, run `set_total` again. Total never decreases, so a smaller
value would otherwise freeze Total until consumption catches up.

## Development

`tests/` holds a config per driver, plus host simulations that run the full logic against a
simulated rotating magnet with noise and thermal drift, and exit non-zero on failure:

```sh
pip install esphome
esphome config tests/test-*.yaml
esphome compile tests/sim-host.yaml
(cd tests && .esphome/build/magdial-sim/.pioenvs/magdial-sim/program)
esphome compile tests/sim-partial-turn.yaml
(cd tests && .esphome/build/magdial-sim-partial/.pioenvs/magdial-sim-partial/program)
```
