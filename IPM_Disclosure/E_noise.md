# ε_noise in Electrical Degradation (D₂)

## What is ε_noise?

**ε_noise** is the **measurement noise floor** — the inherent uncertainty in the switching-time measurement that exists even in a healthy, non-degraded module.

It represents random fluctuations from:
- Timer jitter (clock quantization)
- ADC quantization noise
- Comparator threshold hysteresis
- Electromagnetic interference (EMI)

---

## Physical Origin of Noise Sources

### 1. Timer Capture Jitter

```
PWM edge occurs at precise time t_0

But timer capture resolution is finite:

  Ideal: capture timestamp = t_0
  Reality: captured timestamp = t_0 ± 10 ns (random)

Root cause:
  - MCU clock is quantized (e.g., 10 ns per clock tick on 100 MHz MCU)
  - PWM edge may occur between clock edges
  - Timer samples the edge at next clock boundary
  - Random ±½ clock period = ±5 ns jitter
```

### 2. Comparator Threshold Crossing Uncertainty

```
Shunt voltage crosses overcurrent threshold:

  Ideal: crosses at exact voltage V_threshold
  Reality: threshold has hysteresis, crosses over ~10–20 mV range

Time uncertainty from hysteresis:
  V_shunt rising at ~1V/µs
  20 mV hysteresis × (1 ns per mV) ≈ 20 ns uncertainty
```

### 3. ADC Quantization Noise

```
If using ADC to detect shunt voltage crossing:

  12-bit ADC on 5V range = 5V / 4096 ≈ 1.2 mV per bit

  Time to cross 1.2 mV at 1V/µs = 1.2 ns
  Multiple samples needed to detect crossing with certainty
  Total ADC-related jitter ≈ 5–10 ns
```

---

## Typical ε_noise Value

**For electrical degradation D₂:**

$$\epsilon_{noise} \approx 5\text{–}10 \text{ ns}$$

This represents the **standard deviation** of repeated measurements on the same hardware at the same operating point, with no aging.

### Measurement Verification

```
At manufacturing (healthy module):
  Take 100 switching-time measurements under identical conditions

  Results (histogram):
  ─────────────────────────────
  │ ████
  │ █████
  │ ██████  ← peak at 52 ns
  │ █████
  │ ████
  └────────────────────────────
    48 50 52 54 56  (ns)

  Standard deviation = 4 ns ≈ ε_noise
```

---

## How ε_noise is Used in D₂ Calculation

The degradation score formula:

$$D_2 = \text{clip}\left(\frac{|\delta_2|}{\delta_{2,eol} - \epsilon_{noise}}, 0, 1\right)$$

**The denominator is NOT just δ₂,eol; it's (δ₂,eol − ε_noise)**

### Why Subtract ε_noise?

```
Without subtracting ε_noise:

  δ₂,eol = 30 ns (end-of-life threshold)
  ε_noise = 6 ns (measurement noise)

  Healthy module:
    δ₂ = ±3 ns (noise only, no aging)
    D₂ = |±3| / 30 = 0.1  ← false alarm!

  Problem: Noise alone triggers false "Warning" state

With subtracting ε_noise:

  D₂ = |±3| / (30 - 6) = ±3 / 24 = 0.125  ← still noisy

  Better: Define signal-to-noise threshold
    D₂ > 0.20 = credible signal
    D₂ < 0.20 = noise floor, ignore
```

---

## Signal-to-Noise Ratio (SNR)

The denominator (δ₂,eol − ε_noise) ensures we're measuring **aging signal above the noise floor**.

```
Example:
  ε_noise = 5 ns
  δ₂,eol = 30 ns (EOL threshold)
  Denominator = 30 - 5 = 25 ns

  SNR = signal_span / noise = 25 / 5 = 5:1

  Interpretation: By week 10, aging signal is 5× larger than noise
                  → detectable with confidence
```

---

## Impact of Large ε_noise

If ε_noise is too large, degradation becomes undetectable:

```
Scenario: Poor measurement hardware (100 ns jitter)

  δ₂,eol = 30 ns (true EOL threshold)
  ε_noise = 100 ns (very noisy)
  Denominator = 30 - 100 = -70 ns  ← NEGATIVE!

  Problem: Cannot measure anything; noise > signal

Fix: Improve hardware (better timer, cleaner comparator)
     Or increase observation window (average more measurements)
```

---

## Reducing ε_noise (Best Practices)

### 1. High-Resolution Timer

```
❌ 100 ns resolution timer:
   ε_noise ≈ 50 ns

✓ 10 ns resolution timer (RA6T3):
   ε_noise ≈ 5 ns
```

### 2. Multiple Measurements & Averaging

```
Single measurement:  ε_noise = 8 ns

Average 100 measurements:
   ε_noise → ε_noise / √100 = 8 / 10 = 0.8 ns

Averaging √N reduces noise by factor of √N
```

### 3. Hardware Filtering

```
Shunt voltage through RC low-pass filter:
   ┌─[R]─┐
   │     ├─ to comparator
   └─[C]─┴─ GND

Filter attenuates high-frequency noise (EMI)
Reduces ε_noise by 30–50%
```

### 4. Hysteresis in Comparator

```
Comparator with hysteresis:
   ┌──────────────────
   │      ▲ V_threshold_high
   │    ╱ │
   │  ╱   │ hysteresis window ← reduces sensitivity to noise
   │╱     │
   └──── ▼ V_threshold_low

Larger hysteresis → less noise sensitivity
Trade-off: Reduces measurement precision
```

---

## Practical D₂ Example with ε_noise

```
Manufacturing (Week 0):
  Measured: t_on = 52.0 ± 3 ns (±ε_noise)
  Predicted: t_predicted = 52.5 ns (from LUT)
  Residual: δ₂ = 52.0 - 52.5 = -0.5 ns (within noise)
  ε_noise = 5 ns
  D₂ = |-0.5| / (30 - 5) = 0.5 / 25 = 0.02 ≈ 0.0  ✓ Healthy

Week 5 (moderate aging):
  Measured: t_on = 60.0 ± 3 ns
  Predicted: t_predicted = 52.5 ns (same operating point)
  Residual: δ₂ = 60.0 - 52.5 = 7.5 ns
  D₂ = 7.5 / (30 - 5) = 7.5 / 25 = 0.30  ✓ Caution

Week 10 (severe aging):
  Measured: t_on = 71.0 ± 3 ns
  Predicted: t_predicted = 52.5 ns
  Residual: δ₂ = 71.0 - 52.5 = 18.5 ns
  D₂ = 18.5 / (30 - 5) = 18.5 / 25 = 0.74  ✓ Warning
```

**Note:** Noise (±3 ns) is small relative to aging signal by week 5–10 → detectable

---

## Comparison: D₂ vs. D₃ Noise

| Parameter | ε_noise | EOL Threshold | SNR | Why Different |
|-----------|---------|---------------|-----|--------------|
| **D₂** | 5–10 ns | 20–30 ns | 2–6:1 | Switching happens 10–20 kHz; frequent measurements |
| **D₃** | 50 ns | 300–400 ns | 5–7:1 | Protection latency measured hourly; fewer samples |

**D₃ has larger absolute noise** because:
- Test current is harder to trigger precisely
- ITRIP/VFO edges are slower (nanoSecond-scale vs. 10 ns timescales)
- Fewer measurement opportunities (hourly vs. continuous)

But **SNR is similar** because δ₃,eol is proportionally larger.

---

## Summary: ε_noise Role

| Aspect | Description |
|--------|------------|
| **What it is** | Measurement noise floor (inherent uncertainty) |
| **Typical value (D₂)** | 5–10 ns |
| **Sources** | Timer jitter, comparator hysteresis, ADC quantization, EMI |
| **Why subtract from denominator** | Ensures degradation signal is above noise |
| **If ε_noise too large** | Degradation undetectable; improve hardware |
| **How to reduce** | Higher-resolution timer, averaging, filtering, hysteresis |

---

**Bottom line:** *ε_noise is the measurement uncertainty; subtracting it from the denominator ensures we only report degradation when signal exceeds noise.*
