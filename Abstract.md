# Invention Disclosure: Multi-Physics RUL Prediction Model

## 1. Title
**System and Method for Predicting Remaining Useful Life (RUL) via Multi-Physics Fusion of Electrical and Magnetic Degradation Indicators**

---

## 2. Abstract
This disclosure describes a system and method for predicting the **Remaining Useful Life (RUL)** of an appliance motor by integrating electrical, magnetic, and derived health indicators into a unified statistical degradation model. Unlike traditional diagnostic methods that rely on mechanical vibration or simple threshold-based current monitoring, this invention utilizes a high-sensitivity **Health Index (HI)** derived from the cross-correlation of three distinct domains:

1.  **Electrical Parameters:** Real-time monitoring of winding resistance ($R$), inductance ($L$), and back-EMF ($E_b$) to detect early insulation stress.
2.  **Magnetic Parameters:** Tracking flux linkage decay ($\lambda$) and core material aging—specifically permeability changes ($\mu$)—to quantify irreversible degradation of the magnetic circuit.
3.  **Derived Spectral Indicators:** Analyzing the dynamic shift in the **impedance resonance frequency** ($\Delta f_r$) and $L/R$ time constant deviations as a high-fidelity "fingerprint" of structural and material aging.

The system employs a Bayesian-updated statistical framework to map these multi-physics indicators against historical degradation paths. By correlating the shift in the impedance spectrum with magnetic coercivity decay, the model provides a probabilistic RUL estimate with a defined confidence interval.

---

## 3. Technical Field
*   **Primary:** Predictive Maintenance (PdM)
*   **Secondary:** Electromagnetic Actuators, Material Science, Statistical Modeling

---

## 4. Problem Statement
Current motor health monitoring often focuses on **Point of Failure** detection (e.g., thermal runaway or bearing seizure). These methods lack the sensitivity to detect **Sub-Threshold Degradation**, such as:
*   Subtle core material aging (permeability decay).
*   Minor insulation thinning that hasn't yet caused a short.
*   Permanent magnet weakening due to thermal cycling.

---

## 5. Proposed Solution & Key Indicators
The model utilizes a triple-layered input vector to calculate the appliance's health:

### A. Electrical Domain
*   **Winding Resistance:** Tracked to detect ohmic heating and insulation carbonization.
*   **Back EMF:** Monitored to detect loss of magnetic flux or winding symmetry.

### B. Magnetic Domain
*   **Flux Linkage ($\lambda$):** Directly correlated to torque production efficiency.
*   **Permeability ($\mu$):** Measured via inductance shifts to detect core aging.

### C. Derived/Spectral Domain
*   **Resonance Frequency Shift ($\Delta f_r$):**
    $$f_r = \frac{1}{2\pi\sqrt{LC}}$$
    *A shift in the peak of the impedance plot serves as the primary novelty for detecting simultaneous electrical and magnetic aging.*
*   **L/R Time Constant:** Used to verify consistency between electrical and magnetic states.

---

## 6. Novelty and Advantages
*   **Fusion Logic:** Combines material science (permeability) with circuit theory (impedance plots).
*   **Early Warning:** Detects degradation months before physical vibration or heat occurs.
*   **Noisy Data Resilience:** Multi-parameter cross-correlation reduces "false positives" from sensor noise.

---

## 7. Potential Claims
1.  A method for calculating a motor **Health Index** by correlating magnetic flux decay with electrical impedance shifts.
2.  The use of **Impedance Spectroscopy** peaks as a statistical feature for RUL prediction in consumer appliances.
3.  A predictive framework that adapts to specific **load profiles** by updating degradation slopes ($\alpha$) in real-time.
