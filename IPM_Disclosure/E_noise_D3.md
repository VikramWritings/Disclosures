# ε_noise in Protection-Logic Degradation (D₃)

## What is ε_noise for D₃?

**ε_noise** is the **measurement noise floor** for the ITRIP-to-VFO propagation delay measurement.

It represents random fluctuations in timing that exist even in a healthy, non-degraded protection circuit.

$$\epsilon_{noise} \approx 50 \text{ ns}$$

This is **much larger than D₂'s ε_noise** (5–10 ns) because the D₃ measurement is fundamentally noisier.

---

## Why D₃ Noise is Large (50 ns vs. D₂'s 5–10 ns)

### 1. Slower Signal Edges (Nanosecond-Scale Transitions)

```
D₂ measurement (switching time):
  PWM edge: sharp, clean rise/fall (1–10 ns transition time)
  Current threshold crossing: fast (shunt V rises ~1V/µs)
  Timer capture: captures a crisp edge
  ε_noise ≈ 5 ns

D₃ measurement (protection latency):
  ITRIP comparator: outputs after internal delays (~500 ns)
  VFO driver output: slew-limited by MOSFET (slow edge ~100–300 ns)
  Edge is soft, not crisp:

  ┌──────────────────────────────
  │                    ▲ 5V
  │                  ╱
  │                ╱  ← soft edge (100–300 ns)
  │              ╱
  └────────────╱
              └─ 0V

  Soft edge → uncertain exact crossing point
  ε_noise ≈ 50 ns (half the edge transition time)
```

### 2. Comparator Hysteresis Larger

```
D₂ (shunt comparator):
  Hysteresis: ~10–20 mV (tight)
  At 1V/µs slope: 10–20 ns timing uncertainty

D₃ (ITRIP-to-VFO chain):
  Multiple comparators in series:
    • ITRIP comparator: 30 mV hysteresis → 30 ns uncertainty
    • Debounce logic: 2–4 clock cycles @ 1 MHz = 2–4 µs (but gate is fast, so ~50 ns effective)
    • Level shifter: inherent delay variation ±50 ns

  Combined: ε_noise ≈ 50 ns (root-sum-square of sources)
```

### 3. Test Current Injection Variability

```
D₂ (natural PWM edge):
  ✓ PWM edge is synchronous to MCU clock
  ✓ Timing is deterministic
  ✓ Noise only from measurement hardware

D₃ (deliberate test current injection):
  ✗ Test current must be injected precisely
  ✗ Shunt voltage rises at: dV/dt = (I_test × R_shunt) / (dI/dt)
  ✗ Current rise time is soft, not instantaneous (~10–50 ns)
  ✗ ITRIP threshold crossing is gradual, not crisp
  ✗ Adds ±30–50 ns uncertainty vs. PWM's clean edge
```

### 4. Fewer Measurement Opportunities

```
D₂ (continuous):
  Measured at every switching event (10–20 kHz)
  1000 measurements/second
  Can average to reduce noise by √1000 ≈ 30×

D₃ (sparse):
  Measured once per hour (or once per 1000 cycles)
  1 measurement/hour ≈ 1/3600 per second
  Cannot average effectively
  Noise accumulation is unavoidable
```

---

## Sources of D₃ ε_noise (Detailed Breakdown)

| Source | Contribution | Reason |
|--------|--------------|--------|
| **Timer jitter** | ±20 ns | Dual-capture MTU has ±30 ns resolution; ±½ = ±15 ns |
| **ITRIP threshold crossing** | ±15 ns | Shunt voltage crosses at ~1 V/µs; 15 mV hysteresis |
| **VFO edge slew** | ±25 ns | 100–300 ns transition time; uncertainty in 50% crossing |
| **Comparator delays** | ±10 ns | ITRIP comparator internal delay variation |
| **Level-shifter jitter** | ±15 ns | LVIC-to-HVIC bridge has inherent timing variation |
| **EMI/coupling** | ±10 ns | Noise on VFO signal (open-drain, prone to coupling) |
| **RMS Total** | **~50 ns** | √(20² + 15² + 25² + 10² + 15² + 10²) ≈ 45 ns |

---

## How ε_noise is Used in D₃ Calculation

The degradation score formula:

$$D_3 = \text{clip}\left(\frac{|\delta_3|}{\delta_{3,eol} - \epsilon_{noise}}, 0, 1\right)$$

### Example with Real Numbers

```
Healthy module (week 0):
  Measured: t_ITRIP→VFO = 805 ± 50 ns (±ε_noise)
  Predicted: t_predicted = 805 ns (from LUT)
  Residual: δ₃ = 805 - 805 = 0 ns (no aging)
  ε_noise = 50 ns
  δ₃,eol = 350 ns (50% latency increase = EOL)

  D₃ = |0| / (350 - 50) = 0 / 300 = 0.0  ✓ Healthy

Week 5 (moderate aging):
  Measured: t_ITRIP→VFO = 920 ± 50 ns
  Predicted: t_predicted = 805 ns (same operating point)
  Residual: δ₃ = 920 - 805 = 115 ns

  D₃ = |115| / (350 - 50) = 115 / 300 = 0.38  ✓ Caution

Week 10 (severe aging):
  Measured: t_ITRIP→VFO = 1080 ± 50 ns
  Predicted: t_predicted = 805 ns
  Residual: δ₃ = 1080 - 805 = 275 ns

  D₃ = |275| / (350 - 50) = 275 / 300 = 0.92  ✓ Critical
```

**Note:** By week 5–10, aging signal (115–275 ns) is much larger than noise (50 ns) → detectable

---

## Signal-to-Noise Ratio for D₃

```
SNR = (signal at degradation) / (noise floor)
    = δ_aging / ε_noise
    = (δ₃ at week 10) / ε_noise
    = 275 ns / 50 ns
    = 5.5:1

Interpretation: Aging signal is 5.5× larger than noise
                → Detectable with good confidence

Compare to D₂:
  SNR = 18.5 ns / 6 ns = 3.1:1  (D₂ has lower SNR but more frequent measurements)
```

---

## Why Subtract ε_noise from Denominator?

Without subtracting ε_noise:

```
D₃ = |δ₃| / δ₃,eol = |50| / 350 = 0.14  (noise floor only)

Problem: Healthy module shows D₃ = 0.14 (non-zero)
         False alarm! Operator thinks there's degradation when there isn't

With subtracting ε_noise:

D₃ = |δ₃| / (δ₃,eol - ε_noise) = |50| / (350 - 50) = 50 / 300 = 0.17

Still slightly elevated, but denominator is larger, reducing sensitivity to noise
More importantly: Defines a credibility threshold
  D₃ < 0.20 = likely noise, ignore
  D₃ > 0.30 = signal above noise, credible degradation
```

---

## Consequences of Large ε_noise (50 ns)

### Advantage: Robust to Noise

```
Even with ±50 ns measurement scatter, degradation is detectable:

Week 5:  δ₃ = 115 ± 50 ns
         Range: 65–165 ns
         All values confidently above zero → degradation is real

Week 0:  δ₃ = 0 ± 50 ns
         Range: -50–+50 ns
         Spans zero → noise, not degradation
         D₃ = 0 (healthy)
```

### Disadvantage: Lower Sensitivity

```
Cannot detect very small degradation:

If true degradation is only δ₃ = 30 ns (small):
  D₃ = 30 / 300 = 0.1 (below detection threshold)
  System doesn't flag the small change

Limitation: Cannot measure degradation < ε_noise
           Min detectable: ~50 ns (ε_noise level)
           In practice: need δ₃ > 2×ε_noise = 100 ns to be confident
```

---

## Reducing D₃ ε_noise (Practical Strategies)

### Strategy 1: Improve Hardware (Hard)

```
Current (RA6T3):
  Timer resolution: ±30 ns
  Comparator hysteresis: 15 mV
  Edge slew rate: 100–300 ns
  ε_noise ≈ 50 ns

Better (hypothetical):
  Timer resolution: ±10 ns (use faster MCU)
  Comparator hysteresis: 5 mV (tighter spec)
  Edge slew rate: 20–50 ns (faster driver)
  ε_noise ≈ 15 ns

Trade-off: Cost, power, EMI sensitivity increase
```

### Strategy 2: Averaging (Moderate Difficulty)

```
Current: 1 measurement per hour → ε_noise = 50 ns

Improved: Take 10 measurements per hour
  Average them: ε_noise → 50 / √10 ≈ 16 ns

Result: 3× noise reduction
Trade-off: Slightly more computational overhead, longer testing window
```

### Strategy 3: Software Filtering (Easy)

```
Kalman filter or exponential moving average on D₃ values:

Raw D₃ measurements (noisy):
  Week 0: 0.0, 0.05, -0.02, 0.03, 0.01  (noise floor)

Filtered D₃ (smooth):
  Week 0: 0.0, 0.02, 0.01, 0.02, 0.015  (noise suppressed)

Advantage: Easy implementation in firmware
Trade-off: Adds latency (takes several measurements to converge)
```

### Strategy 4: Longer Observation Window (Simple)

```
Don't report D₃ until week 5–10 when aging >> noise

Healthy (week 0): δ₃ = 0 ± 50 ns → D₃ unknown (< 0.20)
Mid-life (week 5): δ₃ = 115 ± 50 ns → D₃ = 0.38 (credible)
EOL (week 10): δ₃ = 275 ± 50 ns → D₃ = 0.92 (very credible)

By waiting, SNR improves naturally as degradation accumulates
```

---

## D₃ ε_noise vs. D₂ ε_noise Comparison

| Aspect | D₂ (Electrical) | D₃ (Protection) | Why Different |
|--------|-----------------|-----------------|--------------|
| **ε_noise** | 5–10 ns | 50 ns | D₃ edge is softer, fewer measurements |
| **δ at week 10** | 18.5 ns | 275 ns | D₃ degrades 15× more than D₂ |
| **SNR** | 3–4:1 | 5–6:1 | Similar despite different ε_noise |
| **Detection threshold** | δ₂ > 10 ns | δ₃ > 100 ns | D₃ needs larger signal to be credible |
| **Measurability** | High (continuous) | Moderate (sparse) | D₂ has 3600× more measurements |

---

## Practical Impact: When Can We Detect D₃ Degradation?

```
Week 0 (healthy):
  δ₃ ≈ 0 ns
  Noise: ±50 ns
  Signal buried in noise → Cannot detect
  D₃ ≈ 0 (indeterminate)

Week 2–3 (early aging):
  δ₃ ≈ 50 ns
  Noise: ±50 ns
  Signal = Noise (marginal)
  D₃ ≈ 0.17–0.20 (borderline credible)

Week 5 (mid-life):
  δ₃ ≈ 115 ns
  Noise: ±50 ns
  Signal >> Noise (5× larger)
  D₃ ≈ 0.38 (clearly detected) ✓

Week 10 (EOL):
  δ₃ ≈ 275 ns
  Noise: ±50 ns
  Signal >> Noise (5.5× larger)
  D₃ ≈ 0.92 (very confident) ✓
```

**Practical conclusion:** D₃ degradation becomes reliably detectable by week 4–5 (halfway to EOL)

---

## Summary: D₃ ε_noise

| Aspect | Value/Explanation |
|--------|-------------------|
| **ε_noise (D₃)** | ~50 ns (10× larger than D₂) |
| **Root cause** | Soft VFO edge, test-current variability, fewer measurements |
| **Signal at week 10** | ~275 ns (5.5× noise floor) |
| **Detectability** | Good by week 5+ (when δ₃ >> ε_noise) |
| **Denominator in D₃ formula** | (δ₃,eol − ε_noise) ensures signal > noise |
| **Reduction methods** | Better hardware, averaging, filtering, longer window |

---

**Bottom line:** *D₃'s 50 ns noise floor is large, but degradation accumulates fast enough that by mid-life (week 5), the aging signal is 5–6× larger than noise, making D₃ reliably detectable.*
