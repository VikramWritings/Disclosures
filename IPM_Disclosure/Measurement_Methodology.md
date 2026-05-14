# Measurement Methodology for Each Degradation Parameter

## D₁: Thermal Transient Observer

**What you measure:** Junction temperature slope during current pulse

**Step-by-step:**
1. Inject current pulse (100–500 µs, 50–100% rated current) into IPM power stage
2. Sample NTC thermistor via ADC at ≥20 kHz (capture 20–100 samples in transient window)
3. Convert ADC counts to junction temperature using pre-characterized NTC curve
4. Perform linear regression on first 50–100 µs of temperature rise
5. Extract slope: dT_j/dt [K/s]
6. Divide by I² and temperature-dependent reference function: x₁ = (dT_j/dt) / (I² · k_ref(T_amb))
7. Compute degradation: D₁ = clip((x₁ − x₁,₀) / (x₁,eol − x₁,₀), 0, 1)

**Frequency:** Once per 100 power cycles or once per 10 ms (whichever is less frequent)

**Hardware required:** ADC, timer/counter, NTC input

**Flowchart:**

```mermaid
flowchart TD
    A([Start: D1 trigger]) --> B[Inject current pulse<br/>100–500 µs, 50–100% I_rated]
    B --> C[Sample NTC via ADC ≥20 kHz<br/>20–100 samples in transient]
    C --> D[Convert ADC counts → T_j<br/>using NTC curve]
    D --> E[Linear regression<br/>on first 50–100 µs]
    E --> F[Extract slope dT_j/dt]
    F --> G[Normalize:<br/>x1 = dT_j/dt ÷ I² · k_ref T_amb]
    G --> H{First-service<br/>baseline x1,0<br/>in NVM?}
    H -- No --> I[Store x1 as x1,0<br/>Return D1 = 0.0]
    H -- Yes --> J[D1 = clip x1 − x1,0<br/>÷ x1,eol − x1,0<br/>into 0..1]
    J --> K[Persist D1, update dD1/dt]
    K --> L([Return D1])

    classDef ok fill:#e2f7e2,stroke:#27ae60,color:#1d4f2c;
    class L,I ok;
```

---

## D₂: Electrical Latency Observer

**What you measure:** PWM-to-overcurrent propagation delay with operating-point compensation

**Step-by-step:**
1. Issue PWM gate command (rising or falling edge)
2. Capture timestamp t_PWM via high-resolution timer
3. Monitor shunt-resistor voltage (via ADC or external comparator)
4. Capture timestamp t_OC when shunt voltage crosses 90% of expected collector current
5. Calculate latency: t_measured = t_OC − t_PWM [ns]
6. Look up baseline from 81-point LUT at current (V_DC, V_GE, T_j, I_load)
7. Interpolate predicted value: t_predicted = LUT(V_DC, V_GE, T_j, I_load)
8. Calculate residual: δ₂ = t_measured − t_predicted
9. Compute degradation: D₂ = clip(|δ₂| / (δ₂,eol − ε_noise), 0, 1)

**Frequency:** Every switching event (at PWM frequency, typically 10–20 kHz); aggregate over 1000 events for noise rejection

**Hardware required:** Timer-capture peripheral (sub-microsecond precision), ADC or comparator for shunt monitoring

**Flowchart:**

```mermaid
flowchart TD
    A([Start: PWM edge event]) --> B[Capture t_PWM<br/>via timer-capture]
    B --> C[Monitor shunt voltage<br/>ADC or comparator]
    C --> D{Shunt crosses<br/>90% of expected I_C<br/>before timeout?}
    D -- No --> X1[Discard event<br/>FAULT_NO_OC_EDGE]
    D -- Yes --> E[Capture t_OC]
    E --> F[t_measured = t_OC − t_PWM]
    F --> G[Snapshot V_DC, V_GE, T_j, I_load]
    G --> H[Interpolate t_predicted<br/>from 81-pt 4D LUT]
    H --> I[δ2 = t_measured − t_predicted]
    I --> J[Append δ2 to event buffer]
    J --> K{Buffer ≥<br/>1000 events?}
    K -- No --> EndAcc([Wait next PWM edge])
    K -- Yes --> L[Remove outliers<br/>Mean δ2 over buffer]
    L --> M{First-service<br/>baseline δ2,0<br/>in NVM?}
    M -- No --> N[Store δ2 as δ2,0<br/>Return D2 = 0.0]
    M -- Yes --> O[D2 = clip δ2 − δ2,0<br/>÷ δ2,eol − ε_noise<br/>into 0..1]
    O --> P[Persist D2, update dD2/dt]
    P --> Q([Return D2])

    classDef fault fill:#fde2e2,stroke:#c0392b,color:#641e16;
    classDef ok fill:#e2f7e2,stroke:#27ae60,color:#1d4f2c;
    class X1 fault;
    class Q,N,EndAcc ok;
```

---

## D₃: Protection-Logic Latency Observer (Novel)

**What you measure:** ITRIP-to-VFO propagation delay via non-destructive test

**Step-by-step:**
1. Detect shutdown window (converter idle, PWM disabled)
2. Inject deliberate test-current pulse (via PWM modulation or external driver) to trigger overcurrent comparator
3. Monitor shunt voltage; capture timestamp t_ITRIP when voltage crosses ITRIP threshold (0.47–0.50 V)
4. Simultaneously monitor VFO terminal (open-drain output pulled up by external resistor)
5. Capture timestamp t_VFO when VFO transitions from 5V to 0V
6. Calculate latency: t_measured = t_VFO − t_ITRIP [ns]
7. Look up baseline from 18-point LUT at current (V_DD, T_j, R_pull_up)
8. Interpolate predicted value: t_predicted = LUT(V_DD, T_j, R_pull_up)
9. Calculate residual: δ₃ = t_measured − t_predicted
10. Compute degradation: D₃ = clip(|δ₃| / (δ₃,eol − ε_noise), 0, 1)

**Frequency:** Once per hour or once per 1000 power cycles (during shutdown windows only; non-destructive)

**Hardware required:** Timer-capture peripheral (dual-channel for ITRIP and VFO), ADC or comparator, PWM or current-injection capability

**Flowchart:**

```mermaid
flowchart TD
    A([Start: D3 trigger]) --> B{Converter in<br/>shutdown window?}
    B -- No --> X1[Return<br/>FAULT_NOT_IN_SHUTDOWN]
    B -- Yes --> C{V_DC, T_j within<br/>safe test range?}
    C -- No --> X2[Return<br/>FAULT_OUT_OF_RANGE]
    C -- Yes --> D[Snapshot V_DD, T_j, R_pull_up<br/>Arm ITRIP &amp; VFO captures<br/>on shared timer base]
    D --> E[Inject non-destructive<br/>test current pulse<br/>amp ≤ 0.6·I_rated, ≤ 10 µs]
    E --> F{ITRIP edge<br/>captured?}
    F -- No --> X3[Return<br/>FAULT_NO_ITRIP_EDGE]
    F -- Yes --> G{VFO falling edge<br/>captured before<br/>MAX_WAIT_VFO?}
    G -- No --> X4[Log CRITICAL<br/>Return FAULT_VFO_NO_RESPONSE]
    G -- Yes --> H[t_measured = t_VFO − t_ITRIP]
    H --> I{N_SAMPLES_AVG<br/>collected?}
    I -- No --> J[Clear fault latch<br/>Wait settle] --> E
    I -- Yes --> K[Remove outliers MAD<br/>Mean of latencies]
    K --> L[Interpolate t_predicted<br/>from 18-pt 3D LUT]
    L --> M[δ3 = t_measured − t_predicted]
    M --> N{First-service<br/>baseline δ3,0<br/>in NVM?}
    N -- No --> O[Store δ3 as δ3,0<br/>Return D3 = 0.0]
    N -- Yes --> P[D3 = clip δ3 − δ3,0<br/>÷ δ3,eol − ε_noise<br/>into 0..1]
    P --> Q[Update dD3/dt history]
    Q --> R{D3 ≥ 0.8 or<br/>dD3/dt critical?}
    R -- Yes --> S[Raise CRITICAL alert<br/>Mech = Protection-IC aging]
    R -- No --> T[Persist D3, dD3/dt to NVM]
    S --> T
    T --> U([Return D3])

    classDef fault fill:#fde2e2,stroke:#c0392b,color:#641e16;
    classDef ok fill:#e2f7e2,stroke:#27ae60,color:#1d4f2c;
    class X1,X2,X3,X4 fault;
    class U,O ok;
```

---

## Summary: Measurement Comparison

| Parameter | Signal | Frequency | Triggering | Precision | Difficulty |
|-----------|--------|-----------|-----------|-----------|-----------|
| **D₁** | NTC slope | ~100 cycles | Deliberate pulse | ±5% | Easy |
| **D₂** | Switching time | Every event | PWM edge | ±10 ns | Medium |
| **D₃** | Protection latency | Hourly/1k cycles | Test injection | ±50 ns | Hard (novel) |

---

**Key insight:** D₁ and D₂ use deliberate/natural triggering. **D₃ requires non-destructive test-current injection during idle—the innovative enabler.**
