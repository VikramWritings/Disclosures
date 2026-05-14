# k_ref in Thermal Degradation (D₁)

## What is k_ref?

**k_ref(T_amb)** is a **temperature-dependent reference function** that normalizes the NTC thermistor slope against ambient temperature.

It removes the effect of ambient temperature so that only **thermal impedance degradation** remains observable.

---

## The Problem k_ref Solves

The raw NTC slope (dT_j/dt) depends on both:
1. **Thermal impedance Z_th** (what we want to measure—this changes with aging)
2. **Ambient temperature T_amb** (operating condition—noise we want to eliminate)

```
Example:
  Summer (T_amb = 45°C): dT_j/dt = 0.85 K/(W·ms)
  Winter (T_amb = 5°C):  dT_j/dt = 0.72 K/(W·ms)

  Difference = 0.13 K/(W·ms)
  Is this aging? OR just seasonal temperature variation?

  Without k_ref: Cannot tell
  With k_ref: Can isolate the aging signal
```

---

## How k_ref Works

**Definition:**

$$k_{ref}(T_{amb}) = \text{reference slope at end-of-line (healthy state) at temperature } T_{amb}$$

**Normalized metric:**

$$x_1 = \frac{dT_j/dt}{I^2 \cdot k_{ref}(T_{amb})}$$

**Result:** x₁ is independent of both current and ambient temperature.

---

## How k_ref is Established (Calibration)

At **end-of-line manufacturing test**, measure NTC slope at 3–5 reference ambient temperatures:

```
Temperature points:
  T_amb = 25°C  → measure dT_j/dt → k_ref(25) = 0.78 K/(W·ms)
  T_amb = 45°C  → measure dT_j/dt → k_ref(45) = 0.82 K/(W·ms)
  T_amb = 65°C  → measure dT_j/dt → k_ref(65) = 0.86 K/(W·ms)

Interpolation:
  For any T_amb, use linear or polynomial fit:
  k_ref(T_amb) = a + b·T_amb + c·T_amb²
```

**Typical behavior:** k_ref increases ~5% per 20°C (empirical, specific to each IPM family)

---

## Physical Interpretation

**Why does k_ref vary with temperature?**

The reference thermal slope depends on:
- **Heat capacity of thermal stack** (slightly temperature-dependent)
- **Thermal conductivity of materials** (κ decreases with temperature for most materials)
- **Ambient-to-case heat transfer coefficient** (depends on convection, varies with T_amb)

For a healthy module at different ambient temperatures, the transient slope naturally varies slightly. **k_ref(T_amb) captures this variation.**

---

## Practical Example

```
Healthy module (week 0):

Summer operation (T_amb = 45°C):
  Measured: dT_j/dt = 0.85 K/(W·ms)
  Apply normalization: x₁ = 0.85 / (I² · k_ref(45))
                          = 0.85 / (I² · 0.82)
                          = 1.037 / I²

Winter operation (T_amb = 5°C):
  Measured: dT_j/dt = 0.72 K/(W·ms)
  Apply normalization: x₁ = 0.72 / (I² · k_ref(5))
                          = 0.72 / (I² · 0.75)
                          = 0.96 / I²

Note: x₁ values are now roughly comparable (not exact due to measurement noise),
whereas raw slopes differed by ~18%
```

---

## Storage of k_ref

**Where stored:** Non-volatile memory (flash) on host MCU

**Size:** Minimal—5 curve points = 5 × (temperature, value) pairs = ~20–40 bytes

**Format:** Can be stored as:
- Lookup table (discrete points + linear interpolation)
- Polynomial coefficients (if fitting k_ref = a + b·T + c·T²)
- Pre-computed reference values at common temperatures

---

## Typical k_ref Values (Infineon IKCM15L60GD Example)

| T_amb (°C) | k_ref (K/(W·ms)) | Notes |
|------------|------------------|-------|
| 25         | 0.78             | Cool operating |
| 45         | 0.82             | Moderate |
| 65         | 0.86             | Warm |
| 85         | 0.90             | Hot |

**Slope:** ~0.003 per °C (or ~3% increase per 10°C)

---

## Why k_ref is Different from Degradation Threshold

**DO NOT confuse:**

- **k_ref(T_amb)** = Reference slope at healthy state for a given ambient temperature
- **x₁,eol** = End-of-life threshold (how much degradation = 100% aged)

```
Example:
  k_ref(45°C) = 0.82    [reference, healthy]
  x₁,eol = 1.35         [threshold, aged; 50% Z_th increase]

At T_amb = 45°C, summer:
  Healthy module: x₁ = 0.82 → D₁ = 0.0
  Degraded module: x₁ = 1.20 → D₁ = (1.20 - 0.82) / (1.35 - 0.82) = 0.68
```

---

## Summary: k_ref Role

| Aspect | Description |
|--------|-------------|
| **What is it** | Temperature-dependent reference slope (healthy state) |
| **Why it exists** | Normalizes out ambient temperature variation |
| **How established** | Measure NTC slope at 3–5 reference temperatures at EOL |
| **How used** | x₁ = (dT_j/dt) / (I² · k_ref(T_amb)) |
| **Storage** | Non-volatile memory, ~5 points, ~40 bytes |
| **Typical variation** | ±3–5% per 20°C |

---

**Bottom line:** *k_ref removes the thermal environment from the measurement, leaving only the degradation signal visible.*

# K/(W·ms) — Units Explained

## What Does K/(W·ms) Mean?

It's a **normalized thermal slope** with unusual units that combine temperature, power, and time.

Breaking it down:

```
K/(W·ms)
│   │   │
│   │   └─ milliseconds (time)
│   └───── Watts (power)
└───────── Kelvin (temperature)
```

---

## Physical Interpretation

**K/(W·ms)** represents: **temperature rise per unit power per unit time**

$$\text{K/(W·ms)} = \frac{\Delta T_j \text{ [K]}}{\text{Power dissipated [W]} \times \text{time [ms]}}$$

### Example Calculation

```
During current pulse:
  Power dissipated: P = I² · R_on = (10 A)² × 0.5 Ω = 50 W
  Time elapsed: t = 100 ms
  Temperature rise measured: ΔT_j = 4.1 K

Slope: dT_j/dt = 4.1 K / 100 ms = 0.041 K/ms

Normalized: x₁ = 0.041 K/ms / 50 W = 0.00082 K/(W·ms)
```

---

## Why This Unit?

The normalization formula is:

$$x_1 = \frac{dT_j/dt}{I^2 \cdot k_{ref}(T_{amb})}$$

Dimensional analysis:

```
x₁ = [K/ms] / ([A²] × [reference_function])

Since power P = I² · R, and we're dividing by I²:

x₁ ≈ [K/ms] / [W equivalent] = K/(W·ms)
```

**In plain English:** How many Kelvin does the junction temperature rise per Watt of dissipated power per millisecond?

---

## Typical Values (Infineon IKCM15L60GD)

| State | k_ref or x₁ | K/(W·ms) | Interpretation |
|-------|-------------|----------|---|
| Healthy baseline | k_ref(45°C) | 0.78–0.82 | Small slope (good thermal path) |
| Mid-life (50% aged) | x₁ | ~1.00 | 28% increase in slope |
| End-of-life (100% aged) | x₁,eol | ~1.35 | 73% increase in slope |

---

## Real-World Example

```
Healthy module (week 0):
  k_ref = 0.78 K/(W·ms)
  Interpretation: For every Watt of power dissipated,
                  temperature rises 0.78 K per millisecond

  Example: 50 W for 100 ms → ΔT = 0.78 × 50 × 0.1 = 3.9 K

Degraded module (week 10, with voids):
  x₁ = 1.35 K/(W·ms)
  Interpretation: For every Watt of power dissipated,
                  temperature rises 1.35 K per millisecond

  Same example: 50 W for 100 ms → ΔT = 1.35 × 50 × 0.1 = 6.75 K

  Difference: Degraded module rises 73% faster
             (because solder voids block heat flow)
```

---

## Why Use This Weird Unit?

**Advantage:** Normalizes out both load current (I²) and time (ms) in a single measurement

```
Without normalizing:
  Day 1: I = 10 A, ΔT = 4.1 K in 100 ms → slope = 0.041 K/ms
  Day 2: I = 15 A, ΔT = 9.2 K in 100 ms → slope = 0.092 K/ms

  Is Day 2 aged? OR just higher current?
  Cannot tell without dividing by I²

With K/(W·ms) units:
  Day 1: x₁ = 0.041 / (100² × k_ref) = normalized value
  Day 2: x₁ = 0.092 / (225² × k_ref) = same normalized value (if no aging)

  Now comparable across different loads
```

---

## Simple Analogy

Think of **K/(W·ms)** like **"temperature sensitivity"**:

- **High K/(W·ms)** = Temperature sensitive (jumps up fast per Watt) = **Poor thermal path**
- **Low K/(W·ms)** = Temperature stable (rises slowly per Watt) = **Good thermal path**

```
Analogy: Car acceleration

Good brakes (low sensitivity):
  Press brake pedal (1 W effort) → car decelerates slowly (0.1 m/s per second)

Worn brakes (high sensitivity):
  Press brake pedal (1 W effort) → car decelerates quickly (0.3 m/s per second)

Same analogy with thermal:
  Good thermal path (low K/(W·ms)):
    Apply 50 W power → temperature rises 0.78 × 50 = 39 K per second

  Degraded thermal path (high K/(W·ms)):
    Apply 50 W power → temperature rises 1.35 × 50 = 67.5 K per second
```

---

## Summary

| Aspect | Explanation |
|--------|------------|
| **Units** | K/(W·ms) = Kelvin per Watt per millisecond |
| **Meaning** | Temperature rise rate per unit power |
| **Healthy value** | 0.78–0.82 K/(W·ms) (IKCM15L60GD at 45°C) |
| **Degraded value** | 1.35 K/(W·ms) (73% increase = solder voids) |
| **High value** | Indicates poor thermal path (voids, TIM depletion) |
| **Low value** | Indicates good thermal path (healthy) |

---

**Bottom line:** *K/(W·ms) is a normalized thermal sensitivity metric that increases when the thermal path degrades.*
