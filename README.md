# Compensator

### Dual-Hall Analog Filament Buffer Sensor

<img width="1200" height="700" alt="compensator_animation" src="https://github.com/user-attachments/assets/8de09267-140e-4a13-8910-3e2fa24061ce" />

**Compensator** is a dual sensor PCB for a linear filament buffer. It tracks the position of a moving magnet and combines the readings from two Hall sensors into **one analog signal** that a controller can monitor through a single ADC input.

The board also exposes **both individual Hall sensor outputs** for applications that need access to the uncombined signals. Compensator is an analog device.

## How it works

A magnet moves along the PCB (closest to the IC side) as the filament buffer compresses and expands. Two Hall sensors respond to the changing magnetic field, and the onboard analog circuitry produces a single, conditioned position-dependent output (`ADC_OUT`).

In the example AFC setup below:

- **Compressed buffer:** Low ADC reading. AFC reports **Advancing** and increases filament feed.
- **Neutral buffer:** Reading near the configured `neutral_point` (typically `0.5`).
- **Expanded buffer:** High ADC reading. AFC reports **Trailing** and reduces filament feed.

This provides continuous feedback instead of only detecting the two ends of travel. Each Hall sensor also has its own onboard indicator LED.

## Electrical specifications

| Parameter | Specification |
| --- | --- |
| Supply voltage | **5 V DC** |
| Main output | Single analog voltage (`ADC_OUT`) |
| Approximate main output range | **~0.15 V to 3.2 V** |
| Additional outputs | `HALL1_RAW`, `HALL2_RAW` |
| Indicators | Two LEDs, one associated with each Hall sensor |
| Signal processing | Onboard analog circuitry; no sensor-board firmware required |

The **~0.15–3.2 V** range describes the electrical voltage at `ADC_OUT`. The approximately **0–1** readings shown by AFC/Klipper are normalized ADC measurements, **not voltages**. Exact readings depend on magnet position, installation, and the controller's ADC reference.

## Connections

### CN1 — Main connection (3-pin)

| Pin | Signal | Description |
| --- | --- | --- |
| 1 | `GND` | Ground |
| 2 | `ADC_OUT` | Combined analog output to the controller ADC |
| 3 | `VCC` | Regulated **5 V** input |

Connect `ADC_OUT` to an **ADC-capable** input on the controller, not a digital-only input. The controller and Compensator must share ground.

### CN2 — Individual Hall outputs (2-pin)

| Pin | Signal | Description |
| --- | --- | --- |
| 1 | `HALL2_RAW` | Direct Hall sensor 2 output |
| 2 | `HALL1_RAW` | Direct Hall sensor 1 output |

These are the **raw sensor signals**, separate from the combined `ADC_OUT` signal. CN2 does not supply power or ground; the main connector must still be connected.

**Electrical caution:** `ADC_OUT` is nominally approximately 0.15–3.2 V, but **verify that it stays within your MCU ADC input limits** across the full mechanical travel and during startup. 
## AFC / Klipper configuration

The following example configures Compensator with the **AFC Klipper add-on** on an installation that supports the `COMP` buffer type:

```python
[AFC_buffer COMP_buffer1]
type: COMP
adc_pin: Turtle_1:PC4
neutral_point: 0.5
low_point: 0.1
high_point: 0.9
deadband: 0.25
reversed: false
adc_min_val: 0.0
adc_max_val: 1.0
```

- `adc_pin`: ADC-capable MCU input wired to `ADC_OUT`; replace `Turtle_1:PC4` with your own pin.
- `neutral_point`: Target reading for the buffer's neutral position.
- `max_tension` / `max_compression`: Configured operating limits in the AFC buffer implementation.
- `reversed`: Changes the interpreted direction if the physical motion reports backwards.

Add the section to the appropriate AFC configuration file for your installation, then restart Klipper.

## Checking operation

With the board powered and connected, move the buffer by hand and query it from the Klipper console:

```gcode
QUERY_BUFFER BUFFER=COMP_buffer1
```

**Example: compressed buffer**

```text
COMP_buffer1 : Advancing (buffer is compressed - increasing feed)
COMP raw: 0.049  smoothed: 0.048  set_point: 0.50
```

**Example: expanded buffer**

```text
COMP_buffer1 : Trailing (buffer is expanded - reducing feed)
COMP raw: 0.861  smoothed: 0.861  set_point: 0.50
```

A correctly oriented installation should show a smoothly changing reading as the magnet travels between the sensors, with a value near the configured neutral point around the middle of travel. The values above are **example readings**, not required calibration endpoints.

If the reported compressed and expanded states are backwards, check the magnet orientation and use `reversed: true` if appropriate for your setup.

## Notes

- The main output allows a controller to monitor the buffer with **one ADC channel** even though the board contains **two Hall sensors**.
- Individual Hall outputs remain available for custom control systems, debugging, and experimentation.
- Compensator measures the position of the moving magnet; it does not directly measure filament force or tension as a buffer **can not** do this.
- The available travel range and signal direction depend on the mechanical installation and magnet placement.
- The TI DRV5056A4EDBZRQ1 Hall sensors on this PCB, in their given orientation require a strong magnetic field. Use N52 rated magnets for best results.  
