# Invention Disclosure: Differential Mode Classification of Hydraulic Load Degradation in Low Frequency Inverter-Driven Pumps

---

## 1. Title
**System and Method for Pre-Symptomatic Differential Classification of Hydraulic Failure Modes in Inverter-Driven Pumps via Conjunctively-Gated Multi-Channel Current Analysis with Explicit Null Hypothesis and Physics-Aware Online Confounder Compensation**

---

## 2. Abstract
This disclosure describes a non-invasive, firmware-only system for pre-symptomatic differential classification of **hydraulic load degradation** in inverter-driven pumps — discriminating among **cavitation, progressive flow restriction (clog/scale), mechanical wear, and air entrainment** — before any motor stall fault occurs. Specifically optimized for **Low Frequency Inverter** environments in cost-sensitive consumer appliances (dishwashers, washing machines, HVAC pumps), the invention overcomes the reactive limitation of conventional stall detection by **inverting** its minimum-current discard logic and using the operating regions normally thrown away as the primary diagnostic window.

The system extracts diagnostic indicators from four physically independent feature channels of the existing torque-producing current ($I_q$):
1. **Statistical Signature ($C_s$):** Variance and 50–500 Hz spectral energy of $I_q$ during the conjunctively-gated dwell window — responsive to **cavitation**.
2. **Phase-Conditioned Trend ($R_s$):** Slope of mean $I_q$ versus cumulative operating hours, conditioned on (named wash phase × narrow temperature band) — responsive to **progressive flow restriction**.
3. **Speed-Step Dynamic Response ($W_s$):** Effective inertia $J_{\text{eff}}$ and damping $B_{\text{eff}}$ from a second-order load-model fit to $I_q$ transients — responsive to **mechanical wear**.
4. **Intra-Cycle Ripple Slope ($A_s$):** PWM ripple $di/dt$ within active ON intervals as a hydraulic-stiffness probe — responsive to **air entrainment**.

To survive deployment under realistic operating-condition variation, all four channels are passed through a **Physics-Aware Online Confounder Compensator** before classification. This layer is distinguished from prior-art adaptive filtering by four cooperating mechanisms tuned to the multi-channel orthogonality structure:
1. **Persistence-aware update suspension** — updates are paused whenever a channel's residual is large AND sustained across multiple consecutive wash cycles.
2. **Cross-channel consistency** — update suspension is also triggered when the residual *pattern* across multiple channels matches a defined multi-channel signature predicted by failure-mode physics, exploiting the physical correlations between channels under each named failure mode.
3. **Operating-envelope gating** — updates are inhibited whenever the present operating-condition vector falls outside a commissioning-observed subspace, preventing silent extrapolation into untested regions.
4. **Meta-state classifier evidence** — the compensator's own update-suspension-state is provided as additional evidence to the Bayesian classifier, treating sustained suspension as itself shifting the posterior toward non-null modes.

These compensated residuals are fused via a **Multi-Mode Bayesian Classifier with Explicit Null Hypothesis**. The classifier maintains parallel posteriors over named failure modes plus a first-class null no-degradation hypothesis, suppressing notification emission whenever the null dominates. Mode-specific Time-To-Service (TTS) projections drive actionable user notifications ("clean filter", "check inlet supply", "service pump"). By leveraging the dwell regions and PWM artifacts that conventional stall detectors discard — and by adapting in-field through a physics-aware compensator that is structurally prevented from absorbing degradation — the invention provides a **zero-hardware-cost** prognostic for sub-threshold hydraulic failure on existing appliance hardware via firmware update.

This disclosure is the load-side companion to a co-pending motor-side disclosure (Permeability Decay & Edge-Ringing RUL); the two share firmware infrastructure including the physics-aware confounder compensation pattern, but cover physically independent failure domains and use mechanism variants tuned to the physics of each domain.

---

## 3. Background of the Invention

### 3.1 The Reactive "Blind Spot" of Existing Stall Detection
Conventional stall detection in low-cost appliance inverters compares $I_q$, speed-tracking error, or back-EMF residual against a fixed threshold. This presents two diagnostic barriers:
* **Post-Failure Detection:** A stall is declared only after the underlying cause has progressed to user-visible failure.
* **The Minimum-Current Discard:** To suppress false alarms during low-torque dwell phases, stall detectors are gated on a minimum-$I_q$ condition. This explicitly throws away the very operating regions where early-stage degradation signatures are most observable.

### 3.2 The Single-Score Limitation of Current Pump Health Monitoring
Existing pump-current diagnostic patents (US 9,609,997, US 9,872,597, US 7,099,852) produce binary fault flags or single-axis severity scores that conflate cavitation, restriction, and wear. A single score cannot drive mode-specific user notifications.

### 3.3 The Confounder Problem in Real-World Deployment
Even with conjunctive operating-region gating and hand-coded rejection rules, the four feature channels remain perturbed by gradual operating-condition variation:
* **Inlet water temperature** changes fluid viscosity, raising mean $I_q$ — looks like clog progression.
* **Supply voltage drift** at the household level shifts $di/dt$ — looks like air entrainment.
* **Detergent / additive changes** alter fluid properties.
* **Cumulative run time within a wash** warms the windings, shifting $R_s$ and perturbing all four channels.

A generic adaptive filter would absorb degradation drift along with these confounders, defeating the entire prognostic purpose. What is needed is a compensator that is *structurally* prevented from absorbing degradation — by exploiting the multi-channel orthogonality of the diagnostic system and the coupling to a downstream null-hypothesis classifier.

### 3.4 The Necessity of the Invention
There is a critical need for a method that (a) extracts diagnostic information from the very dwell regions current designs discard, (b) discriminates among physically distinct failure modes using only existing inverter sensing, (c) projects mode-specific time-to-service with explicit confidence bounds, and (d) compensates for in-field operating-condition variation through a layer specifically architected to absorb only the reversible variation while leaving degradation drift to propagate as classifier evidence.

---

## 4. Detailed Description: Conjunctive Operating-Region Gating
The system inverts established stall-detector design practice by deliberately operating within the window that minimum-$I_q$ gating excludes:

$$\text{diagnostic\_active} = (\text{phase\_id} = \text{CIRC\_DWELL}) \wedge (\omega_{\text{cmd}} < \omega_{\text{dwell\_th}})$$

This conjunction distinguishes the diagnostic dwell from coincidentally-low-$I_q$ operating points. Within the gated window, the small mean-$I_q$ baseline maximizes fractional resolvability of degradation signatures.

---

## 5. Detailed Description: Channel 1 — Variance and Spectral Energy (Cavitation)
Cavitation produces stochastic torque pulsations as vapor cavities collapse on the impeller, coupling into $I_q$ via $T = (3/2) p \lambda_{pm} I_q$.

### 5.1 Step-by-Step Algorithm
1. **Capture:** $I_q$ samples within the gated dwell window at the current control loop rate (8–16 kHz).
2. **Compute:** Variance $\sigma^2_{I_q}$ and band-limited spectral energy $E_{\text{cav}} = \int_{50}^{500} |\widehat{I_q}(f)|^2 df$.
3. **Compensate:** Pass through the physics-aware compensator (Section 9) before scoring.
4. **Score:** $C_s = \sqrt{\text{var\_ratio} \cdot \text{band\_ratio}}$ from compensated residuals.

---

## 6. Detailed Description: Channel 2 — Phase × Temperature Conditioned Trend (Restriction)
1. **Log:** End-of-phase mean $I_q$ logged to NV storage indexed by `(phase_id, temp_band)`, with band width ≤ 10 °C.
2. **Compensate:** Logged value replaced by its residual against the compensator's expected value.
3. **Estimate:** RLS slope of residual history vs. cumulative operating hours.
4. **Score:** $R_s$ = confidence-weighted slope.

---

## 7. Detailed Description: Channel 3 — Speed-Step Dynamic Response (Wear)
1. **Trigger:** Commanded speed transitions where $|\Delta \omega_{\text{cmd}}|$ exceeds threshold within one tick.
2. **Capture:** $I_q$ response over post-transition window.
3. **Fit:** Second-order load model yielding $J_{\text{eff}}$ and $B_{\text{eff}}$.
4. **Compensate:** Express as residuals against compensator expected values.
5. **Score:** $W_s$ from compensated residuals.

---

## 8. Detailed Description: Channel 4 — Intra-Cycle Ripple as Hydraulic Probe (Air Entrainment)
PWM ripple slope $di/dt$ governed by $V_{dc} - V_{bemf} \approx L_{\text{eff}} \frac{di}{dt} + R_s \cdot i$.

### 8.1 The Temporal-Resolution Carve-Out
Motor-side inductance $L$ drifts on a timescale of thousands of operating hours. Hydraulic stiffness reflected through the rotor changes within seconds. Therefore within-dwell deviations in $di/dt$ are necessarily attributable to load-side state changes, not motor-side aging.

### 8.2 Step-by-Step Algorithm
1. **Capture:** $di/dt$ at sub-microsecond resolution within active PWM ON intervals during the gated dwell.
2. **Compare:** Against model expectation derived from a baseline updated only at completed-cycle granularity.
3. **Compensate:** Pass deviation through the compensator (Section 9).
4. **Integrate:** Compensated fractional deviation into Air-Entrainment Score $A_s$.

---

## 9. Detailed Description: Physics-Aware Online Confounder Compensator
A small recursive linear estimator runs per feature channel, learning the normal mapping between operating conditions and channel values from running history alone. The compensator is distinguished from prior-art adaptive filtering by four cooperating mechanisms.

### 9.1 The Model
For each channel $y \in \{C_s, R_s, W_s, A_s\}$:
$$\hat{y}(\mathbf{x}) = \theta_0 + \theta_1 T_{water} + \theta_2 V_{dc} + \theta_3 \omega_{cmd} + \theta_4 t_{run}$$
The residual $e = y - \hat{y}$ is what the Bayesian classifier sees, not the raw channel value.

### 9.2 Mechanism 1 — Persistence-Aware Update Suspension
The compensator suspends its own coefficient updates whenever both:
1. **Magnitude:** $|e| > k \cdot \sigma_{resid}$
2. **Persistence:** Condition (1) holds for $\geq N$ consecutive wash cycles

This exploits the temporal asymmetry between transient confounders (water temperature varies cycle-to-cycle, voltage sags are episodic) and persistent degradation (clogs progress, wear accumulates while the underlying root cause remains).

### 9.3 Mechanism 2 — Cross-Channel Consistency
The compensator additionally suspends updates whenever the residual *pattern across multiple channels* matches a defined physical signature of one of the named failure modes. Specifically:
* **Cavitation signature:** $C_s$ residual elevated AND $A_s$ residual modestly elevated AND others quiet
* **Clog signature:** $R_s$ residual sustained-positive AND eventually $C_s$ residual rising as cavitation onset begins
* **Wear signature:** $W_s$ residual sustained AND no others elevated
* **Air-entrainment signature:** $A_s$ residual elevated AND $C_s$ residual modestly elevated

When today's residual vector matches one of these signatures within a defined similarity threshold, all four compensators freeze. This leverages the orthogonality-by-physical-construction of the four channels: confounders typically affect one channel in isolation; real degradation produces correlated multi-channel patterns predicted by the physics of each named mode.

This cross-channel coupling is unique to multi-channel diagnostic systems with physics-grounded orthogonality and is not present in single-channel adaptive filtering, generic concept-drift literature, or motor-control RLS prior art.

### 9.4 Mechanism 3 — Operating-Envelope Gating
During commissioning, the firmware records the convex hull (or covariance ellipsoid) of operating-condition vectors $\mathbf{x} = (T_{water}, V_{dc}, \omega_{cmd}, t_{run})$ encountered. Each cycle, the firmware checks whether today's $\mathbf{x}$ falls inside the commissioning-validated envelope:
* **Inside the envelope** → normal compensator update (subject to Mechanisms 1 and 2)
* **Outside the envelope** → updates suspended; residual flagged as low-confidence to the classifier

This prevents silent absorption of degradation-like behavior caused by extrapolation into untested regions of operating space — a known failure mode of generic adaptive filters that do not condition update gating on input-space coverage.

### 9.5 Mechanism 4 — Meta-State as Classifier Evidence
The compensator emits two outputs per cycle: the residual value AND an **update-suspension-state flag** indicating whether the compensator has been frozen for the past $M$ cycles. The Bayesian classifier's likelihood model includes a term that says: when sustained suspension is observed across multiple compensator instances, increase the prior on non-null hypotheses by a defined factor.

Sustained suspension is itself diagnostic — it means the compensators detect something they cannot explain via operating-condition variation. Generic adaptive filters discard their meta-state; the present invention exposes it as evidence channeled to the downstream classifier.

### 9.6 Architectural Separation from Control Loop
Parameter estimates $\boldsymbol{\theta}$ of the compensator are explicitly NOT used by any closed-loop control function of the inverter. Only the residual and the suspension-state flag are provided to the prognostic system. This carves out the entire body of online-RLS-for-motor-control prior art (sensorless position estimation, MPC parameter identification, etc.) which uses similar machinery for control rather than for diagnostic-side residual generation.

---

## 10. Detailed Description: Multi-Mode Bayesian Classifier with Explicit Null
The four compensated residuals AND the four compensator suspension-state flags feed a multi-hypothesis Bayesian recursive filter operating in three asynchronous loops:
* **High-Speed Loop:** Captures $I_q$ and $di/dt$; computes per-pulse and per-interval features.
* **Compensation Loop:** Per-cycle update of $\boldsymbol{\theta}$ with all four mechanisms; emits residuals + meta-state.
* **Prognostic Loop:** Updates posterior over $\{\text{NULL}, \text{CAVITATION}, \text{RESTRICTION}, \text{WEAR}, \text{AIR\_ENTRAINMENT}\}$ at end of each wash cycle.

### 10.1 The Null Hypothesis as a Structural Element
Unlike prior-art Bayesian/fuzzy pump classifiers (Liu 2015; Perovic; Luo 2015), the **null no-degradation hypothesis is a first-class hypothesis** with its own prior, likelihood, and posterior.

### 10.2 Notification Suppression Gate
Notification emission is conditional on **both**:
* $P(\text{null}) \leq \tau_{\text{null}}$ AND
* $P(\text{argmax non-null}) \geq \tau_{\text{conf}}$

---

## 11. Comparative Advantage

| Feature | US 9,609,997 | US 9,872,597 | US 7,099,852 | VFF RLS / Generic | This Invention |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Output type** | Binary flag | Binary flag | Severity score | N/A | **Posterior + TTS** |
| **Operating window** | Generic low-load | Drain phase | Steady state | N/A | **Conjunctive (phase ∧ speed)** |
| **Channels** | 1 | 1 | FFT bins | N/A | **4 orthogonal** |
| **Null hypothesis** | Implicit | Implicit | Implicit | N/A | **Explicit, suppression-gated** |
| **Confounder handling** | Static rules | None | None | Magnitude-based | **Persistence + Cross-Channel + Envelope-Gated** |
| **Decay-Absorption Risk** | N/A | N/A | N/A | High | **Structurally prevented** |
| **Use of Compensator State** | N/A | N/A | N/A | Discarded | **Exposed as classifier evidence** |
| **Sensors** | Existing | Existing | External CTs | N/A | **Software-only** |

---

## 12. Specific Claims
1. A method for detection of fluid-side **air entrainment** in a pump driven by a Low Frequency Inverter, comprising capturing within active PWM ON intervals during a conjunctively-gated circulation-dwell phase a slope $di/dt$ of a phase current; comparing said slope to a model expectation that is insensitive to motor-side inductance drift on timescales longer than the dwell phase; and integrating fractional deviation into an Air-Entrainment Score, without addition of any sensor beyond those required for closed-loop motor control.

2. A method for **differential classification of hydraulic failure modes** comprising extracting four feature channels selected for orthogonal physical response (statistical, trend, dynamic, intra-cycle); applying a multi-hypothesis Bayesian classifier maintaining parallel posteriors over named modes including an **explicit null no-degradation hypothesis** as a first-class hypothesis; and emitting mode-specific notifications **only when the null posterior is below a defined threshold AND a non-null mode posterior exceeds a defined confidence threshold**.

3. A method for **conjunctively-gated hydraulic state observation** in a pump operating concurrently with stall-detection logic, wherein a diagnostic operating interval is identified by the conjunction of (state-machine reporting circulation-dwell phase) AND (commanded speed below dwell threshold), said interval being the very interval **excluded by the concurrent stall detector's minimum-current gating condition**; and computing variance and 50–500 Hz spectral energy metrics referenced to commissioning baselines, exclusively within said interval.

4. A method as in Claim 2, wherein the trend feature channel is computed as a **recursive-least-squares slope** of mean torque-producing current versus cumulative operating hours, conditioned on the joint index (named wash phase identifier × fluid temperature band of width not exceeding **10 °C**), eliminating fluid viscosity drift as a confounder of clog progression detection.

5. A method as in Claim 2, comprising a **physics-aware online recursive linear estimator** per feature channel that:
   (a) suspends coefficient updates whenever the absolute residual exceeds a defined magnitude threshold for at least $N$ consecutive wash cycles;
   (b) additionally suspends coefficient updates whenever the residual pattern across the plurality of feature channels matches one of a defined set of mode-specific multi-channel signatures, said signatures comprising correlations between channels predicted by the physics of each named failure mode of Claim 2;
   (c) suspends coefficient updates whenever the present operating-condition vector falls outside a commissioning-observed subspace beyond a defined extrapolation tolerance; and
   (d) emits an update-suspension-state signal that is provided as additional evidence to the multi-hypothesis classifier of Claim 2, said state signal increasing the posterior probability of non-null hypotheses when sustained suspension is observed; wherein parameter estimates of said estimator are not used by any closed-loop control function of the inverter, and wherein only the residual and the suspension-state signal are provided to the multi-hypothesis classifier.

---

## 13. Prior Art and Novelty Analysis

### 13.1 Known Prior Art
This invention acknowledges:
* **Variance-Based Cavitation Detection ([US 9,609,997](https://patents.google.com/patent/US9609997B2/en)):** Variance threshold for cavitation flag in dishwashers.
* **Mean-Current Threshold Clog Detection ([US 9,872,597](https://patents.google.com/patent/US9872597B2/en)):** Mean current vs. fixed threshold for filter clog.
* **Neural-Network Pump Fault Classification ([US 7,099,852](https://patents.google.com/patent/US7099852B2/en) / [US 7,389,278](https://patents.google.com/patent/US7389278B2/en)):** NN classification of FFT features into pump faults.
* **FFT Cavitation Detection in Dishwasher ([US 8,277,571](https://patents.google.com/patent/US8277571B2/en)):** FFT to distinguish cavitation harmonics.
* **Bayesian / Fuzzy Pump-Fault Classifiers (Perovic; Luo 2015; [Liu 2015](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0125703)):** Always declare a most-likely fault mode; no first-class null.
* **RLS Load-Parameter Identification ([EP 1113102](https://patents.google.com/patent/EP1113102B1/en); [WO 2004/097099](https://patents.google.com/patent/WO2004097099A1/en)):** Single-cycle load measurement.
* **PWM-Based Inductance Identification ([US 11,656,286](https://patents.google.com/patent/US11656286B2/en)):** Motor-side inductance for control.
* **Optical Air-Entrainment Measurement ([US 4,164,137](https://patents.google.com/patent/US4164137A/en)):** Optical particle counter; fluid-contacting.
* **Variable Forgetting Factor RLS ([US 7,225,215](https://patents.google.com/patent/US7225215B2/en); academic VFF literature):** Adaptive forgetting based on residual *magnitude*, symmetric in sign, single-channel.
* **Compensation Learning for Diagnostics ([US 10,578,045](https://patents.google.com/patent/US10578045B2/en)):** Drift detection followed by *command-side* compensation in engine control loop.
* **Concept Drift in Condition Monitoring (academic literature):** Detect-then-adapt strategies; generic, not domain-specific.

### 13.2 Distinguishing Novelty of the Present Invention

1. **Re-Purposing of PWM Ripple as a Hydraulic-Stiffness Probe (Claim 1):** Within a state-machine-gated dwell window where motor-side inductance cannot have drifted, with a temporal-resolution carve-out distinguishing from US 11,656,286 motor-side use of the same signal.

2. **Explicit Null Hypothesis with Structural Suppression Gate (Claim 2):** First-class null hypothesis with structural suppression — not present in prior-art pump-fault classifiers.

3. **Conjunctive Operating-Region Inversion (Claim 3):** Diagnostic window defined as the very interval the concurrent stall detector excludes — structural, not coincidental.

4. **Joint Phase × Narrow-Temperature-Band Trend Conditioning (Claim 4):** Eliminates phase-mix and viscosity drift as confounders.

5. **Persistence-Aware Update Suspension (Claim 5(a)):** Update suspension conditioned on residual persistence across consecutive wash cycles. Distinguishes from VFF RLS prior art, which uses magnitude-based forgetting variation rather than residual-persistence-based update suspension.

6. **Cross-Channel Consistency Coupled to Failure-Mode Physics (Claim 5(b)):** Update suspension is additionally conditioned on the residual pattern across multiple channels matching one of a defined set of *mode-specific* multi-channel signatures predicted by failure-mode physics. This leverages the four-channel orthogonality-by-physical-construction (Claim 2) as a discriminator between confounders (which typically affect one channel in isolation) and degradation (which produces correlated multi-channel patterns specific to each failure mode). No single-channel adaptive filter, generic concept-drift method, or motor-control RLS approach has this multi-channel-pattern-matched coupling.

7. **Operating-Envelope Gating (Claim 5(c)):** The compensator maintains an explicit representation of the commissioning-observed operating-condition subspace and suspends updates whenever the present operating point falls outside it. This prevents silent absorption of degradation-like behavior caused by extrapolation. Generic adaptive filters do not condition update gating on input-space coverage.

8. **Compensator Meta-State as Diagnostic Evidence (Claim 5(d)):** The compensator's update-suspension-state flag is exposed as an additional evidence channel to the Bayesian classifier, treating sustained suspension as itself shifting the posterior toward non-null hypotheses. Generic adaptive filters discard their meta-state.

9. **Architectural Separation from Control Loop:** Parameter estimates are explicitly NOT used by any closed-loop control function; only residuals and suspension-state flags are exposed to the prognostic system. This carves out the entire body of online-RLS-for-motor-control prior art (sensorless position estimation, MPC parameter identification, online inertia estimation) which uses similar machinery for control rather than for diagnostic-side residual generation.

10. **Pre-Symptomatic, Software-Only, Consumer-Deployable Operation:** The combination of conjunctive gating, conditional baselining, four-channel synergy, physics-aware confounder compensation, and ≤2 KB NV footprint defines a system that produces actionable mode-specific notifications during fault-free operation, on existing appliance hardware via firmware update, with no added sensors. No prior-art reference, alone or in combination, teaches this combination.
