# Technical Specification: Hydraulic Load Observer with Physics-Aware Confounder Compensation
## Target Platform: Renesas RA6T3 (Cortex-M33)

This document describes the implementation of a firmware-based observer to detect **hydraulic load degradation** in an inverter-driven dishwasher pump. This is achieved by extracting four physically independent feature channels from the existing torque-producing current ($I_q$), gated on a conjunctive operating-region condition that inverts the discard logic of the concurrent stall detector, and passing all four channels through a **Physics-Aware Confounder Compensator** that is structurally prevented from absorbing degradation as if it were operating-condition variation.

This observer is a **co-pending companion to the Permeability Decay Observer** running on the same MCU. The two observers share DAQ infrastructure and the same compensator pattern, but probe physically independent failure domains and use mechanism variants tuned to each domain's physics.

---

## 1. System Overview
Since hydraulic load degradation occurs over hundreds-to-thousands of operating cycles and is corrupted by water temperature, supply voltage, command profile, and run-time effects, the firmware compares compensated residuals against a "Phase-Conditioned Baseline" stored in non-volatile memory and indexed by `(wash_phase_id, temperature_band)`. The compensator subtracts known operating-condition effects from the four raw channels, leaving only the unexplained drift to propagate as evidence to the Bayesian classifier.

| Component             | Specification                                       |
| :-------------------- | :-------------------------------------------------- |
| **MCU**               | Renesas RA6T3                                       |
| **Control Frequency** | 16 kHz (PWM & FOC Loop)                             |
| **Sample Rate**       | 16 kHz $I_q$, ~1 MHz windowed $di/dt$ during PWM ON |
| **Channels**          | 4 (Cs, Rs, Ws, As)                                  |
| **Compensator**       | Per-channel RLS + Physics-Aware Suspension Logic    |
| **Conditioning Vars** | $T_{water}$, $V_{dc}$, $\omega_{cmd}$, $t_{run}$    |
| **Failure Modes**     | Cavitation, Restriction, Wear, Air-Entrainment, Null |
| **NV Footprint**      | ≤ 2 KB (commissioning baselines + 4 RLS states)     |

---

## 2. Mathematical Logic
The observer extracts four feature channels, each responsive to a different failure mode through different physics. All four pass through the compensator before fusion.

### 2.1 Channel 1 — Statistical (Cavitation)
Cavitation produces stochastic torque pulsations. With $T = (3/2) p \lambda_{pm} I_q$:
$$\sigma^2_{I_q} \approx \frac{\text{Var}(T)}{((3/2)\,p\,\lambda_{pm})^2}$$
Band-limited spectral energy augments the variance:
$$E_{\text{cav}} = \int_{50}^{500} |\widehat{I_q}(f)|^2 \, df$$

### 2.2 Channel 2 — Trend (Restriction)
Recursive least-squares fit of mean $I_q$ versus cumulative operating hours, conditional on `(phase_id, temp_band)`:
$$\bar{I_q}(h) \approx m \cdot h + b$$

### 2.3 Channel 3 — Dynamic Response (Wear)
Second-order load model fit to $I_q$ transients following commanded speed steps:
$$J_{\text{eff}}\,\dot\omega + B_{\text{eff}}\,\omega + T_{\text{load}} = \frac{3}{2} p \lambda_{pm} I_q$$

### 2.4 Channel 4 — Intra-Cycle Ripple (Air Entrainment)
During the active PWM ON interval:
$$V_{dc} - V_{bemf} \approx L_{\text{eff}}\frac{di}{dt} + R_s \cdot i$$
$L_{\text{baseline}}$ updated only at completed-cycle granularity, so within-dwell deviations are necessarily load-side.

### 2.5 Confounder Model
The compensator maintains a 5-coefficient linear model per channel:
$$\hat{y}(\mathbf{x}) = \theta_0 + \theta_1 T_{water} + \theta_2 V_{dc} + \theta_3 \omega_{cmd} + \theta_4 t_{run}$$
The residual $e = y - \hat{y}$ is what the Bayesian classifier sees, not the raw channel value.

### 2.6 Persistence Suspension Rule
Updates suspended when **both** hold:
1. $|e| > k \cdot \sigma_{resid}$
2. Condition (1) holds for $\geq N$ consecutive wash cycles

### 2.7 Cross-Channel Signature Suspension Rule
Each named failure mode has a defined **multi-channel signature** predicted by physics. Updates also suspended when today's residual vector matches one of these signatures within similarity threshold:
- **Cavitation:** $C_s$ elevated, $A_s$ modestly elevated, others quiet
- **Restriction:** $R_s$ sustained-positive, $C_s$ slowly rising
- **Wear:** $W_s$ sustained, others quiet
- **Air entrainment:** $A_s$ elevated, $C_s$ modestly elevated

This leverages the four-channel orthogonality so confounders (typically affect one channel) are distinguished from real degradation (produces correlated multi-channel patterns).

### 2.8 Operating-Envelope Gating
Compensator records the commissioning-observed envelope of $\mathbf{x}$ vectors. Each cycle:
- Inside envelope → normal RLS update (subject to rules above)
- Outside envelope → updates suspended; residual flagged as low-confidence

To account for the lack of dedicated hydraulic sensors, the algorithm uses the **Conjunctive Dwell Gate** to ensure features are computed in the operating window where signal-to-noise is highest and the concurrent stall detector is blind:
```
diagnostic_active = (phase_id == CIRC_DWELL) && (w_cmd < W_DWELL_TH)
```

---

## 3. Firmware Architecture

### A. Peripheral Configuration (RA6T3)
* **GPT (Timer)**: Center-Aligned PWM at 16 kHz. GPT triggers the ADC at carrier peaks (Channels 1–3) and at PWM rising/falling edges (Channel 4 windows).
* **ADC12**: Phase current sampling. Double Data Register at $t=0$ and $t=T/2$ for ripple cancellation on Channels 1–3. For Channel 4, oversampled bursts at sub-microsecond resolution.
* **DTC (Data Transfer Controller)**: Moves ADC samples into ring buffers without CPU load.
* **CMSIS-DSP**: Goertzel for 50–500 Hz energy on Channel 1, second-order LS fit on Channel 3, per-channel RLS update for compensator.
* **CAN-FD**: Receives `(phase_id, w_cmd, T_fluid_NTC, V_dc)` from the appliance MCU.

### B. Signal Processing Flow
1. **Gate**: Apply conjunctive dwell gate `(phase_id == CIRC_DWELL) && (w_cmd < W_DWELL_TH)`.
2. **Capture**: Within the gated window, sample $I_q$ and $di/dt$ at appropriate rates.
3. **Extract**: Compute four raw channel scores ($C_s$, $R_s$, $W_s$, $A_s$).
4. **Reject**: Discard the interval if confounders (door open, $V_{dc}$ transient, heater active) are detected.
5. **Compensate**: Pass each raw score through its physics-aware compensator (Section 4).
6. **Classify**: Update multi-mode Bayesian posterior with explicit null hypothesis using compensated residuals + meta-state flag.
7. **Suppress or Emit**: Notification only if `P(null) < τ_null AND P(mode) > τ_conf`.
8. **Store**: Persist to **Data Flash** at end of each wash cycle.

---

## 4. Implementation Example (C-Code)

```c
/*
 * RA6T3 Hydraulic Load Observer with
 * Physics-Aware Confounder Compensator
 */

#include "hal_data.h"
#include "arm_math.h"

#define DWELL_SPEED_TH_RPM   800.0f
#define TEMP_BAND_WIDTH_C    10.0f
#define CAV_BAND_LO_HZ       50.0f
#define CAV_BAND_HI_HZ       500.0f
#define LAMBDA_RLS           0.99f
#define SUSP_PERSIST_N       8        // consecutive cycles
#define NULL_DOMINANCE_TH    0.7f
#define MODE_CONFIDENCE_TH   0.7f

typedef enum {
    MODE_NULL = 0,           // Explicit null hypothesis - first-class
    MODE_CAVITATION,
    MODE_RESTRICTION,
    MODE_WEAR,
    MODE_AIR_ENTRAINMENT,
    MODE_COUNT
} HydraulicMode_t;

typedef struct {
    float theta[5];          // RLS coefficients per channel
    float P[25];             // 5x5 covariance row-major
    float resid;             // last residual
    int   persist_count;     // cycles with |e| > threshold
    bool  in_envelope;       // operating-condition envelope check
    bool  suspended;         // current suspension state
    int   suspension_count;  // cycles in suspension
} CompensatorState_t;

typedef struct {
    CompensatorState_t comp[4];      // one per channel
    float posterior[MODE_COUNT];     // includes explicit null
    bool  meta_state_flag;           // sustained multi-channel suspension
} HydraulicHealth_t;

static HydraulicHealth_t g_health;

/* Conjunctive operating-region gate (the inversion) */
static inline bool dwell_gate_active(uint8_t phase_id, float w_cmd) {
    return (phase_id == PHASE_CIRC_DWELL)
        && (w_cmd < DWELL_SPEED_TH_RPM);
}

/* Operating-envelope check (ellipsoidal) */
static bool inside_envelope(const float *x, const float *x_center,
                            const float *envelope_radii) {
    float dist_sq = 0;
    for (int i = 0; i < 4; i++) {
        float d = (x[i] - x_center[i]) / envelope_radii[i];
        dist_sq += d * d;
    }
    return dist_sq <= 1.0f;
}

/* Cross-channel signature match - returns true if today's residual
   pattern matches one of the named failure-mode signatures */
static bool matches_mode_signature(const float r[4]) {
    /* Cavitation: r[0] high, r[3] modest, r[1] r[2] quiet */
    if (r[0] > 1.5f && r[3] > 0.5f && r[1] < 0.3f && r[2] < 0.3f) return true;
    /* Restriction: r[1] sustained positive, r[0] rising */
    if (r[1] > 1.0f && r[0] > 0.4f && r[2] < 0.3f && r[3] < 0.3f) return true;
    /* Wear: r[2] sustained, others quiet */
    if (r[2] > 1.5f && r[0] < 0.3f && r[1] < 0.3f && r[3] < 0.3f) return true;
    /* Air entrainment: r[3] high, r[0] modest */
    if (r[3] > 1.5f && r[0] > 0.5f && r[1] < 0.3f && r[2] < 0.3f) return true;
    return false;
}

/* Per-channel compensator update */
void compensator_update(CompensatorState_t *c, float y_meas,
                        const float *x, bool envelope_ok,
                        bool mode_signature_active) {
    /* Compute residual */
    float y_pred = c->theta[0];
    for (int i = 0; i < 4; i++) y_pred += c->theta[i+1] * x[i];
    c->resid = y_meas - y_pred;

    /* Persistence tracking */
    if (fabsf(c->resid) > THRESHOLD_K * sigma_resid(c)) c->persist_count++;
    else c->persist_count = 0;

    /* Suspension decision - any of three rules can suspend */
    c->suspended = (!envelope_ok) ||
                   (c->persist_count >= SUSP_PERSIST_N) ||
                   mode_signature_active;

    if (c->suspended) {
        c->suspension_count++;
        return;  /* DO NOT update theta or P */
    }

    /* Normal RLS update */
    rls_update(c->theta, c->P, x, c->resid, LAMBDA_RLS);
    c->suspension_count = 0;
}

/* Run all four compensators per cycle */
void run_compensators(const float raw_scores[4], const float *op_cond) {
    bool env_ok = inside_envelope(op_cond,
                                  COMMISSIONING_CENTER,
                                  COMMISSIONING_RADII);

    /* Compute provisional residuals first for cross-channel check */
    float prov_resid[4];
    for (int ch = 0; ch < 4; ch++) {
        float pred = g_health.comp[ch].theta[0];
        for (int i = 0; i < 4; i++)
            pred += g_health.comp[ch].theta[i+1] * op_cond[i];
        prov_resid[ch] = raw_scores[ch] - pred;
    }
    bool sig_active = matches_mode_signature(prov_resid);

    /* Update each compensator with all three rules in play */
    for (int ch = 0; ch < 4; ch++) {
        compensator_update(&g_health.comp[ch], raw_scores[ch],
                           op_cond, env_ok, sig_active);
    }

    /* Meta-state flag: sustained multi-compensator suspension */
    int suspended_count = 0;
    for (int ch = 0; ch < 4; ch++)
        if (g_health.comp[ch].suspension_count > SUSP_PERSIST_N)
            suspended_count++;
    g_health.meta_state_flag = (suspended_count >= 2);
}

/* Multi-mode Bayesian classifier with explicit null + meta-state */
static const float MODE_WEIGHTS[MODE_COUNT][4] = {
    /*                     Cs    Rs    Ws    As   */
    [MODE_NULL]            = { 0.0f, 0.0f, 0.0f, 0.0f },
    [MODE_CAVITATION]      = { 1.0f, 0.0f, 0.0f, 0.3f },
    [MODE_RESTRICTION]     = { 0.0f, 1.0f, 0.0f, 0.0f },
    [MODE_WEAR]            = { 0.0f, 0.0f, 1.0f, 0.0f },
    [MODE_AIR_ENTRAINMENT] = { 0.4f, 0.0f, 0.0f, 1.0f },
};

static void update_mode_posterior(void) {
    float r[4];
    for (int i = 0; i < 4; i++) r[i] = g_health.comp[i].resid;

    float total = 0.0f;
    for (int m = 0; m < MODE_COUNT; m++) {
        float L = (m == MODE_NULL)
            ? compute_null_likelihood(r)
            : compute_mode_likelihood(r, MODE_WEIGHTS[m]);

        /* Meta-state flag boosts non-null likelihoods */
        if (g_health.meta_state_flag && m != MODE_NULL) L *= 1.5f;

        g_health.posterior[m] *= L;
        total += g_health.posterior[m];
    }
    if (total > 1e-9f)
        for (int m = 0; m < MODE_COUNT; m++)
            g_health.posterior[m] /= total;
}

/* End-of-cycle prognostic with notification suppression */
HydraulicMode_t prognostic_loop_end_of_cycle(float cumulative_hours) {
    update_mode_posterior();

    HydraulicMode_t best = MODE_NULL;
    for (int m = 1; m < MODE_COUNT; m++)
        if (g_health.posterior[m] > g_health.posterior[best])
            best = (HydraulicMode_t)m;

    /* STRUCTURAL SUPPRESSION GATE - emit only if BOTH hold */
    if (g_health.posterior[MODE_NULL] > NULL_DOMINANCE_TH) return MODE_NULL;
    if (g_health.posterior[best] < MODE_CONFIDENCE_TH)     return MODE_NULL;

    float tts_hours = project_tts_for_mode(best, cumulative_hours);
    emit_user_notification(best, tts_hours);
    return best;
}
```

---

## 5. Decision Matrix (Mode-Specific Notifications)

| Dominant Mode (P > 0.7) | Meta-State Flag | User-Facing Notification         | Firmware Action                      |
| :---------------------- | :-------------- | :------------------------------- | :----------------------------------- |
| `NULL`                  | Clear           | *(suppressed)*                   | Continue normal operation            |
| `RESTRICTION`           | Set             | "Clean filter"                   | Log code; project TTS; advise user   |
| `CAVITATION`            | Set             | "Check water inlet supply"       | Log code; project TTS; advise user   |
| `AIR_ENTRAINMENT`       | Set             | "Ensure pump is fully primed"    | Run prime cycle; recheck after 1 cycle |
| `WEAR`                  | Set             | "Schedule pump service"          | Log code; field-service notification |

The meta-state flag is what differentiates "operating-condition-driven drift" (compensators adapt freely) from "real degradation" (compensators freeze and the Bayesian classifier sees mounting evidence). This structural separation is what the prior art lacks.

---

## 6. VS Code Integration Tips
* Use the **C/C++ Extension** for IntelliSense on RA6T3 registers and the CMSIS-DSP `arm_*_f32` family.
* Enable **Markdown Preview** (`Ctrl+Shift+V`) to view this spec formatted alongside the Permeability Decay Observer spec.
* Use **Renesas Configurator** to ensure GPT triggers for Channels 1–3 (PWM-peak ADC) and Channel 4 (PWM-edge oversampled bursts) do not collide on the ADC sequencer.
* Recommended: build a unit-test harness that replays recorded $I_q$ traces from real fault conditions (clogged filter, primed-vs-unprimed, scale-deposited) against the four channels — this validates that confounders (cold-water cycles, voltage sags) are absorbed by the compensator while real degradation patterns trigger the cross-channel signature rule and the meta-state flag.
