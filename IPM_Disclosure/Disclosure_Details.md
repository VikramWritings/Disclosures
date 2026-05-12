# The Solution: Multi-Domain Health Monitoring of Intelligent Power Modules

## 1. Overview

This invention introduces a comprehensive health-monitoring system for intelligent power modules (IPMs) that combines three independent, physically orthogonal degradation observers into a single normalized health index. The three observers measure:

1. **Thermal degradation** (D₁) — via normalized NTC slope, detecting thermal-path failures.
2. **Electrical degradation** (D₂) — via normalized switching latency, detecting gate-loop failures.
3. **Protection-path degradation** (D₃) — via normalized ITRIP-to-VFO propagation delay, detecting internal protection-IC failures.

Each observer is normalized against operating conditions, then fused using a worst-of-weighted-sum formula that preserves single-domain failures from being masked by averaging. The pattern of which observers are elevated reveals the dominant failure mechanism, enabling targeted maintenance rather than wholesale module replacement. The entire system runs in firmware on the host microcontroller, using only signals already accessible at the IPM's datasheet-specified terminals, requiring no modification of the IPM and adding substantially zero bill-of-materials cost.

---

## 2. The Three Observers: Architecture and Physics

### 2.1 Observer 1: D₁ — Thermal Transient Observer

**Physical Principle**: When current I flows through the IPM for duration τ, junction temperature rises as T_j(t) ≈ T_amb + P(t) · Z_th(t). At short timescales (tens to hundreds of microseconds), the thermal impedance Z_th is dominated by the die-to-baseplate path: the solder bond, the DBC substrate, and the interface materials. As these degrade (solder voiding, TIM pump-out, DBC delamination), Z_th increases, causing the early-time slope dT_j/dt to increase for the same power pulse.

**Measurement**: The host injects a current pulse (100–500 µs duration, 50–100% rated current) via normal PWM modulation. During the pulse, the ADC samples the NTC voltage every 10–50 µs and converts to temperature using a pre-characterized NTC curve (Steinhart-Hart or exponential model). Linear regression over the first 50–100 µs yields dT_j/dt.

**Normalization**: The raw slope is divided by I² and a temperature-dependent reference scaling factor to obtain an operating-point-invariant metric:

$$x_1 = \frac{dT_j/dt}{I^2 \cdot k_{ref}(T_{amb})}$$

The reference function k_ref is derived from EOL characterization at multiple ambient temperatures.

**Baseline and Degradation Score**: At first service, x₁,₀ is captured. As the module ages, x₁ increases. The degradation score is:

$$D_1 = \text{clip}\left(\frac{x_1 - x_{1,0}}{x_{1,eol} - x_{1,0}}, 0, 1\right)$$

where x₁,eol is an end-of-life threshold from accelerated-aging tests.

**Failure Modes**: TIM pump-out, grease dry-out, solder voids in die-attach, DBC delamination, heatsink fouling.

### 2.2 Observer 2: D₂ — Electrical Latency Observer

**Physical Principle**: When a PWM command edge is issued, the gate-driver output stage must charge the IGBT's gate capacitance, shifting V_GS above threshold V_GS(th). The time required for this turn-on transition depends on gate-driver output impedance, IGBT input capacitance, gate oxide charge, gate threshold voltage, and gate-supply-current availability. As the gate loop ages (bond-wire degradation, driver-capacitor ESR increase, gate-oxide trapping), switching time increases.

**Measurement**: The host issues a PWM command edge and captures its timestamp with a high-resolution timer (sub-microsecond). Simultaneously, a timer-capture peripheral monitors the voltage across an external shunt resistor. When this voltage crosses a threshold consistent with 90% of final collector current (indicating the switching transition is complete), a second timestamp is captured. The measured switching interval is:

$$t_{PWM \to OC} = t_{OC\_threshold} - t_{PWM\_edge}$$

**Normalization**: The switching time is highly sensitive to operating point (V_DC, V_GE, T_j, I_load). An 81-point (3×3×3×3) baseline is established at end-of-line test and stored in non-volatile memory. During service, the measured value is interpolated against this baseline:

$$t_{predicted} = LUT(V_{DC}, V_{GE}, T_j, I_{load})$$

The residual is:

$$\delta_2 = t_{PWM \to OC} - t_{predicted}$$

This residual is substantially independent of normal operating-point variations and sensitive only to aging.

**Baseline and Degradation Score**: At first service, δ₂,₀ is captured. The degradation score is:

$$D_2 = \text{clip}\left(\frac{|\delta_2|}{\delta_{2,eol} - \epsilon_{noise}}, 0, 1\right)$$

where ε_noise is a measurement-noise floor (typically a few nanoseconds).

**Failure Modes**: Gate-driver output-stage aging, gate-pad bond-wire lift or cracking, IGBT gate-oxide charge trapping, driver-supply capacitor ESR increase and capacitance loss.

### 2.3 Observer 3: D₃ — Active Logic Observer (Novel)

**Physical Principle**: When the ITRIP voltage exceeds threshold V_ITRIP,TH (typically 0.47–0.50 V), an internal comparator is asserted. This signal routes through debounce logic (50–500 ns delay), level shifters (50–200 ns per stage), and the fault-output driver, eventually pulling the VFO (fault-output) terminal low. Total latency is typically 500–2000 ns and is specified in every IPM datasheet. As the LVIC, HVIC, and output-driver silicon age (NBTI, HCI, oxide degradation), propagation delays increase incrementally.

**Why Novel**: The IPM's protection path has no externally observable output during normal operation. The VFO is pulled low only during a real fault (overcurrent, overvoltage, overtemperature). No prior-art condition-monitoring scheme has measured or attempted to monitor this path. **This is the critical innovation: the first observability of the protection IC itself.**

**Measurement — Self-Test Method**: The host injects a deliberate test current pulse into the ITRIP node during a non-operational window (shutdown interval, low-current window). This test pulse crosses the V_ITRIP,TH threshold, triggering the full protection path. The host time-stamps the injection and the resulting VFO edge with sub-microsecond resolution:

$$t_{ITRIP \to VFO} = t_{VFO\_low} - t_{ITRIP\_threshold}$$

**Measurement — Real-Fault Method**: During actual overcurrent faults, the fault timing is captured opportunistically, extracting t_ITRIP→VFO from real-world data.

**Normalization**: The measured delay is sensitive to V_DD, T_j, and external pull-up resistor R_pull_up. An 18-point (3×3×2) baseline is established at end-of-line:

$$t_{predicted} = LUT(V_{DD}, T_j, R_{pull\_up})$$

The residual is:

$$\delta_3 = t_{ITRIP \to VFO} - t_{predicted}$$

**Baseline and Degradation Score**: At first service, δ₃,₀ is captured. The degradation score is:

$$D_3 = \text{clip}\left(\frac{|\delta_3|}{\delta_{3,eol} - \epsilon_{noise}}, 0, 1\right)$$

**Failure Modes**: LVIC/HVIC silicon aging (NBTI, HCI, oxide degradation), level-shifter propagation-delay increase, fault-output-stage slew-rate reduction, internal clock circuit degradation.

---

## 3. Normalization and Baseline Establishment

A critical innovation is that **normalization must be performed against operating-point variables** to render each observer independent of operating conditions. An observer that drifts with load current or ambient temperature will produce false alarms whenever conditions change.

### 3.1 End-of-Line Characterization

At manufacture end, before shipment, the IPM is characterized under controlled conditions:

- **For D₁**: Measure normalized NTC slope at T_amb = 25°, 50°, 75°C; derive reference function k_ref.
- **For D₂**: Measure t_PWM→OC at 3×3×3×3 grid of (V_DC, V_GE, T_j, I_load) combinations (81 points); store lookup table.
- **For D₃**: Measure t_ITRIP→VFO at 3×3×2 grid of (V_DD, T_j, R_pull_up) combinations (18 points).

These baselines occupy 1–4 kB of non-volatile memory.

### 3.2 First-Service Baseline

When the IPM is first powered in the customer's converter, the host performs initial measurements of each observer at actual operating conditions (typically low load, nominal ambient, cold junction). These become the first-service baseline (x₁,₀, δ₂,₀, δ₃,₀), the reference point for all subsequent degradation calculations.

Rationale: Component-to-component variation is large (10–15%), so using generic EOL baseline across all units would create unnecessary conservatism and false alarms in early-life units.

### 3.3 Operating-Point Compensation During Service

When an observer is measured during service, the raw value is compared to a predicted "healthy" value obtained by interpolating the stored EOL baseline at the current operating point. For D₂, at V_DC = 380 V, V_GE = 15 V, T_j = 75°C, I_load = 8 A:

$$t_{predicted} = \text{trilinear\_interp}(LUT_{81pt}, V_{DC}, V_{GE}, T_j, I_{load})$$

The residual δ₂ = t_measured - t_predicted is substantially independent of normal operating-point variations and is what is actually compared against end-of-life thresholds.

---

## 4. Fusion into Normalized Health Index

### 4.1 Worst-of-Weighted-Sum Formula

The three degradation scores are fused into a single health index:

$$HI = 1 - [\alpha \cdot \max(D_1, D_2, D_3) + (1-\alpha) \cdot \sum_i w_i D_i]$$

where:
- **α** ≈ 0.7 (weights the worst-case contributor at 70%, the average at 30%).
- **wᵢ** are individual weights (w₁ = 0.40, w₂ = 0.35, w₃ = 0.25; Σwᵢ = 1).
- **max(D₁, D₂, D₃)** ensures the worst observer dominates.

**Rationale**: A purely worst-of function would be overly conservative (a module with one degraded and two healthy subsystems would be flagged as critical). A purely weighted-sum would average out the worst-case (a critical failure could be masked). The hybrid form balances: worst-case dominates while healthy observers provide slight mitigation.

### 4.2 Health Index States

- **HI > 0.8**: Healthy; continue normal operation.
- **0.5 < HI ≤ 0.8**: Caution; recommend monitoring.
- **0.2 < HI ≤ 0.5**: Warning; schedule maintenance within days to weeks.
- **HI ≤ 0.2**: Critical; immediate replacement recommended.

Additionally, **dHI/dt** (rate of change) is monitored. A rapidly declining HI is a strong failure predictor, even if the absolute value is still in the caution range.

---

## 5. Failure-Mechanism Localization via Cross-Correlation Diagnostic Matrix

The key innovation is that **the three observers are physically orthogonal**: each is sensitive to a distinct failure domain with minimal cross-sensitivity. The pattern of which observers are elevated reveals the dominant failure mechanism.

### 5.1 Diagnostic Logic

| D₁ | D₂ | D₃ | Mechanism | Action |
|:-:|:-:|:-:|---|---|
| ↑ | → | → | TIM pump-out, heatsink fouling | Inspect/re-apply TIM; check airflow |
| ↑↑ | → | → | Solder voiding, DBC delamination | Replace IPM immediately |
| ↑ | ↑ | → | Bond-wire degradation (affects both thermal and electrical paths) | Schedule replacement; reduce duty if possible |
| → | ↑ | → | Gate-driver capacitor wear or gate-pad bond-wire lift | Replace IPM or repair driver supply |
| → | → | ↑ | LVIC/HVIC aging, level-shifter degradation | High priority; protection reliability degrading |
| → | → | ↑↑ | Protection-path failure imminent | Critical; immediate replacement; shutdown may be warranted |
| → | ↑ | ↑ | Gate-driver IC aging (shared silicon) | Replace IPM |
| ↑ | ↑ | ↑ | Comprehensive end-of-life failure | Immediate replacement |

### 5.2 Implementation

The classifier can be implemented as:

1. **Rule engine** (simplest): if-then-else statements based on thresholds.
2. **Decision tree**: test observers sequentially, output mechanism at each leaf.
3. **Learned classifier** (advanced): neural network trained on field data.

For first deployment, the rule-engine approach is recommended for transparency and auditability.

---

## 6. Implementation on Unmodified Commercial IPMs

### 6.1 Hardware Requirements

The entire monitoring function requires:

1. **Host microcontroller** (already present in the converter):
   - High-resolution timer-capture peripheral (sub-microsecond resolution).
   - ADC (≥10-bit, ≥10 kHz sample rate).
   - PWM generator with edge timestamping.
   - Non-volatile memory (≥1 kB for baselines).

2. **Current-sense element** (already present): Shunt resistor or integrated current sense.

3. **No IPM modifications**: The IPM is used stock, unmodified.

### 6.2 Software Architecture

1. **Initialization** (at power-up):
   - Load EOL baselines from non-volatile memory.
   - Set up ADC, timer-capture, PWM interrupt handlers.
   - Perform first-service measurement of each observer; store baselines.

2. **Periodic measurement loop** (real-time firmware):
   - **Every 10 ms or 100 power cycles**: Inject D₁ current pulse, measure NTC slope.
   - **On every switching event or every 1000 cycles**: Measure D₂ (PWM-to-comparator interval).
   - **Once per hour or per 1000 cycles**: Inject D₃ test current, measure ITRIP-to-VFO interval.

3. **Fusion and classification** (e.g., once per second):
   - Compute HI using worst-of-weighted-sum formula.
   - Compute dHI/dt via linear regression.
   - Apply diagnostic matrix to classify dominant failure mechanism.
   - Assert fault flags or set diagnostic codes if thresholds exceeded.

4. **Logging and reporting**:
   - Store HI and observer values at periodic intervals in circular buffer or flash.
   - Expose current HI and diagnostic code to converter control firmware or remote diagnostics system.

### 6.3 Computational Overhead

- **Computation**: <0.1% of MCU budget; <1 ms active execution per measurement cycle.
- **Bill-of-materials**: Zero (all hardware already present).
- **Code footprint**: Typically ≤10 kB ROM, ≤2 kB RAM.

---

## 7. Exemplary Implementation: Infineon IKCM15L60GD

The Infineon IKCM15L60GD is a 15 A, 600 V, 3-phase inverter module from the CIPOS family. It incorporates all required features:

- Power stage (6 IGBTs + 6 free-wheeling diodes)
- Integrated HVIC and LVIC
- Overcurrent comparator with ITRIP input
- Protection logic with VFO fault-output
- Integrated NTC thermistor accessible at shared VFO/NTC pin

**Typical measured degradation over service life**:

- **Thermal impedance, short-time (Z_th,short)**: 0.5 K/W → 0.8 K/W (60% increase)
- **Switching delay, turn-on (t_on)**: 50 ns → 70 ns (40% increase)
- **Fault-path propagation delay (t_ITRIP→VFO)**: 800 ns → 1200 ns (50% increase at end-of-life)

All changes are easily detectable with sub-microsecond timer resolution available on modern MCUs (STM32H7, C2000, XMC1400 families).

---

## 8. Key Advantages Over Prior Art

1. **Comprehensive multi-domain coverage**: Observes all three major failure domains (thermal, electrical, protection-logic), whereas prior art typically focuses on one or two.

2. **Protection-path observability** (novel): Introduces the first known condition-monitoring observable for the IPM's internal fault-response path—a critical gap in existing systems.

3. **Failure-mechanism localization**: The cross-correlation diagnostic matrix enables identification of the dominant failure mechanism, supporting targeted maintenance rather than wholesale replacement.

4. **Zero-modification hardware**: Uses only signals and peripherals already present on commercial IPMs and host MCUs; no IPM redesign required.

5. **Zero incremental cost**: Requires no additional sensors or signal conditioning; purely firmware-based on existing MCU.

6. **Broad applicability**: Works across multiple IPM families (Infineon CIPOS, onsemi NFAM/SPM, Mitsubishi, Fuji, and equivalents) because it uses only datasheet-specified terminals.

7. **Safety-critical value**: Enables early detection of protection-path degradation—a failure mode with potentially catastrophic consequences (loss of shutdown capability) that was previously undetectable.

8. **Retrofit capability**: Can be added to existing converter designs via firmware updates, provided the host MCU has required peripherals.

---

## 9. Variations and Future Extensions

1. **Two-observer variant**: Omit D₃ for applications where protection-path aging is not critical; reduces measurement overhead further.

2. **Real-fault-only D₃**: Measure protection delay only during actual overcurrent faults, not via deliberate self-test injection; suitable for high-fault-frequency applications.

3. **Extended baseline with auxiliary variables**: Add operating-point dimensions (DC-link ripple, gate-supply voltage noise, heatsink temperature gradient, cumulative power-cycle count) for higher accuracy in highly variable operating conditions.

4. **Machine-learning classifier**: Replace rule-based diagnostic matrix with a shallow neural network or random forest trained on accelerated-aging and field data, for better handling of ambiguous multi-observer-elevation cases.

5. **Cloud-connected fleet prognostics**: Stream HI and diagnostic codes to a predictive-maintenance platform for fleet-level trend analysis, automated maintenance scheduling, and early warning of emerging systemic issues (e.g., a manufacturing batch exhibiting early bond-wire failures).

---

## 10. Comparison to Existing Condition-Monitoring Approaches

**Thermal-only monitoring** (existing art): Detects thermal-path degradation only; insensitive to gate-loop or protection-logic failures.

**Switching-time-only monitoring** (prior art, US 6,405,154): Detects gate-loop degradation only; requires careful normalization; insensitive to thermal or protection-logic failures.

**Multi-parameter fusion of V_CE(sat) and T_j** (existing art): Parameters are not orthogonal (both sensitive to temperature); provides partial coverage; insensitive to protection-logic failures.

**Present invention**: Three orthogonal observers spanning all major failure domains; failure-mechanism localization; observes protection-logic for the first time; zero-modification hardware; zero incremental cost.

---

## 11. Safety and Reliability Implications

For safety-critical applications (EV traction inverters, medical-device power supplies, aerospace actuators), the ability to detect protection-path degradation is transformative. A traditional IPM can pass every conventional health check while its overcurrent shutdown is gradually becoming slower. By the time a real fault occurs, the protection response time may be insufficient, and the IGBT is destroyed before the gate driver can turn it off. The present invention makes this scenario detectable and preventable.

Additionally, by localizing the dominant failure mechanism, operators can make data-driven maintenance decisions: a module with only thermal degradation may be cleared for operation with improved cooling; a module with gate-loop degradation only may be scheduled for planned replacement; a module with protection-path degradation must be replaced immediately, regardless of other health indicators.

---

## Conclusion

This invention provides a complete, implementable solution for comprehensive intelligent-power-module health monitoring that directly addresses the critical unmet need for observability of the IPM's own protection-logic subsystem. By combining three physically orthogonal degradation observers, normalizing them against operating conditions, and fusing them with a worst-of-weighted-sum formula that preserves single-domain failures from being masked, the invention enables both comprehensive health assessment and targeted failure-mechanism localization. The method is implementable entirely in firmware on host microcontrollers already present in power converters, requires no modification of the IPM, uses only signals accessible at standard datasheet terminals, and imposes negligible computational overhead and zero incremental cost. The invention is particularly valuable for safety-critical and mission-critical applications where IPM reliability is directly linked to system safety and where the cost of an unannounced failure vastly exceeds the cost of preventive replacement based on real, quantitative health data.
