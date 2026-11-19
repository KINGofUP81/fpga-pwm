# FPGA PWM Demo (Zybo)

Mixed-language PWM playground for the Digilent Zybo board. It combines several PWM generators behind a switch-selected top level:

- linear duty sweep
- sine-table modulation
- RGB rainbow fading
- a servo-style output path
- a standalone servo generator module that is present but currently unused

> Status: source-level review only. There are no Vivado project files or build scripts in this tree, and nothing here has been compiled or simulated as part of this documentation pass.

## Origin / attribution

This PWM lab appears to be course material built around the FPGA design style of **Pong P. Chu’s** *FPGA Prototyping by ... Examples* books. The `sin_pwm.vhd` source contains the comments `-- Start from slides` and `-- End from slides`, indicating that the sine-ROM section was supplied as lecture material. The Zybo XDC is Digilent’s generic Zybo master constraints file. No author name or license file was present in the original sources.

---

## Directory layout

```text
fpga-pwm/
├── constraints/
│   └── zybo-master.xdc
├── sim/
│   ├── linear_pwm_tb.sv
│   ├── pwm_enhanced_tb.sv
│   ├── pwm_rgb_tb.sv
│   ├── rainbow_pwm_tb.sv
│   ├── sin_pwm_tb.sv
│   └── top_pwm_tb.sv
└── src/
    ├── linear_pwm.sv
    ├── pwm_enhanced.sv
    ├── pwm_rgb.sv
    ├── rainbow_pwm.sv
    ├── servo.vhd
    ├── sin_pwm.vhd
    └── top_pwm.sv
```

---

## Modules

| Module | Language | Purpose |
|---|---|---|
| `pwm_enhanced` | SystemVerilog | Core PWM generator: prescaler counter, duty counter, registered PWM output. |
| `linear_pwm` | SystemVerilog | Sweeps duty from `0` to `2**resolution` at a slow rate. |
| `sin_pwm` | VHDL | Sine lookup table generated at elaboration with `IEEE.MATH_REAL`; feeds `pwm_enhanced`. |
| `pwm_rgb` | SystemVerilog | Single-channel fader with `offset → rise → high → fall → low` states and a configurable delay. |
| `rainbow_pwm` | SystemVerilog | Three `pwm_rgb` instances with delays `4`, `0`, and `2` for R/G/B phase shifts. |
| `servo` | VHDL | Generic servo PWM generator using real-valued timing generics. **Not instantiated by `top_pwm`.** |
| `top_pwm` | SystemVerilog | Instantiates all generators and selects the output with `sw`. |

---

## Top-level operation

`top_pwm.sv` uses the following switch priority, highest first:

| Switch | Selected output | Visible effect |
|---|---|---|
| `sw[0]` | `linear_out` | `rgb[0]` gets a linearly ramping PWM signal; other RGB channels off. |
| `sw[1]` | `sin_out` | `rgb[2]` gets a sine-modulated PWM signal; other RGB channels off. |
| `sw[2]` | `r_out/g_out/b_out` | Full rainbow fader on `rgb[2:0]`. |
| `sw[3]` | `servo_out_reg` | Currently a sine-derived PWM signal routed to `servo_out`; RGB off. |
| none | — | RGB off. |

Reset is active-high and asynchronous at the top level.

---

## Timing parameters

Defaults in `top_pwm.sv`:

| Parameter | Value | Meaning |
|---|---:|---|
| `resolution` | 8 | PWM duty-counter width. |
| `dvsr` | 4882 | PWM prescaler divisor. |
| `grad_thresh` | 2_500_000 | Clock cycles per gradient/duty update step. |

With the 125 MHz Zybo clock:

```text
f_pwm      = 125 MHz / ((dvsr + 1) × 2**resolution)
           ≈ 125 MHz / (4883 × 256)
           ≈ 100 Hz

f_gradient = 125 MHz / grad_thresh
           = 50 Hz
```

So the carrier is about 100 Hz while the duty envelope updates at about 50 Hz.

`sin_pwm` uses `sin_thresh = grad_thresh`, so it advances one sine-table entry every 2.5 M clocks, i.e. 50 Hz; a full 256-entry sine table takes about 5.12 seconds.

---

## Board connections

From `constraints/zybo-master.xdc`:

| Signal | Pin(s) | Notes |
|---|---|---|
| `clk` | K17 | `sysclk`, 125 MHz, 8 ns period |
| `rst` | K18 | `btn[0]` |
| `sw[0]` | G15 | Linear mode |
| `sw[1]` | P15 | Sine mode |
| `sw[2]` | W13 | Rainbow mode |
| `sw[3]` | T16 | Servo mode |
| `rgb[0]` | V16 | LED6 red |
| `rgb[1]` | F17 | LED6 green |
| `rgb[2]` | M17 | LED6 blue |
| `servo_out` | T14 | JD connector pin |

---

## Simulation

The testbenches use a 4 ns half-period, modeling a 125 MHz clock.

Example with Vivado XSim (after creating a project and adding the sources):

```tcl
add_files src/*.sv
add_files src/*.vhd
add_files -fileset sim_1 sim/*_tb.sv

# Select one testbench at a time
set_property top pwm_enhanced_tb [get_filesets sim_1]
launch_simulation
```

Available testbenches:

| Testbench | DUT |
|---|---|
| `pwm_enhanced_tb.sv` | Bare PWM core with fixed duty `64`. |
| `linear_pwm_tb.sv` | Linear sweep. |
| `sin_pwm_tb.sv` | Sine PWM. |
| `pwm_rgb_tb.sv` | Three `pwm_rgb` channels with delays `4`, `0`, `2`. |
| `rainbow_pwm_tb.sv` | `rainbow_pwm` wrapper. |
| `top_pwm_tb.sv` | Full top level, cycles through every switch mode. |

The testbenches currently rely on waveform inspection; they contain no assertions or automatic pass/fail checks.

---

## Known issues

These are source-level findings, not simulation results.

1. **The servo output is not driven by `servo.vhd`.**  
   `top_pwm.sv` instantiates a second `sin_pwm` for `servo_out_reg`. That produces a roughly 100 Hz, 0–100% duty waveform, not a valid 50 Hz / 1–2 ms servo pulse.

2. **`servo_out` is not cleared in non-servo modes.**  
   The top-level sequential block only assigns `servo_out` in reset and in the `sw[3]` branch. In other modes it holds its previous value. Add an `else servo_out <= 1'b0;`.

3. **`gradient_pulse` is not reset.**  
   `linear_pwm.sv` and `pwm_rgb.sv` assign `gradient_pulse` in every non-reset branch, but leave it uninitialized in the reset branch. It can start as `X` in simulation.

4. **`sin_pwm.vhd:89` uses an overflowing conversion.**  
   `to_unsigned(2**resolution, resolution)` tries to represent 256 in 8 bits. Use `resolution + 1` for that compare, or compare directly against the MSB of `duty_sin`.

5. **Possible mixed-language generic-type concern.**  
   `top_pwm.sv` passes a SystemVerilog `[31:0]` vector parameter to the VHDL `std_logic_vector(31 downto 0)` generic `dvsr` in `sin_pwm`. Verify this elaborates cleanly in your Vivado version; if not, change the VHDL generic to `integer` or add an explicit conversion wrapper.

6. **`servo.vhd` is standalone/unwired.**  
   It has no generic defaults and requires `clk_hz`, `pulse_hz`, `min_pulse_us`, `max_pulse_us`, `step_count`, and a `position` input. A top-level position source (switches, counter, or test pattern) would be needed to use it.

---

## Suggested cleanup work

- Replace the sine-derived servo path with a real 50 Hz servo controller (either `servo.vhd` or a dedicated SystemVerilog module).
- Drive `servo_out` low in all non-servo modes.
- Reset `gradient_pulse` in `linear_pwm` and `pwm_rgb`.
- Fix the `to_unsigned` width in `sin_pwm`.
- Add assertion-based testbenches that check period, duty range, and mode selection.
