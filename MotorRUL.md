# Invention Disclosure: Multi-Physics RUL Prediction via Sensorless Impedance Spectroscopy

## 1. Title
**System and Method for Sensorless Remaining Useful Life (RUL) Prediction using Inverter-Driven Multi-Physics Health Indicators**

---

## 2. Abstract
This disclosure describes a system for predicting the **Remaining Useful Life (RUL)** of an electric motor by leveraging existing power electronics to perform **In-Situ Impedance Spectroscopy**. The method utilizes the motor’s **Pulse Width Modulation (PWM) inverter** as a high-frequency signal generator, eliminating the need for external diagnostic hardware. By injecting a high-frequency carrier signal into the stator windings, the system extracts a dynamic **Impedance Plot** from existing current sensors. 

The core innovation lies in the fusion of three degradation domains: 
1. **Electrical (Winding resistance and L/R constants)**
2. **Magnetic (Flux linkage and permeability $\mu$)**
3. **Derived Spectral Shifts (Resonance frequency $\Delta f_r$)**

By correlating the horizontal shift and vertical flattening of the impedance resonance peak with **core material aging** and **insulation fatigue**, a Bayesian statistical model generates a probabilistic RUL estimate. This enables appliances to self-diagnose material-level degradation during standard operation.

---

## 3. Technical Field
*   **Primary:** Predictive Maintenance (PdM) / Prognostics and Health Management (PHM)
*   **Secondary:** Variable Frequency Drives (VFD), Magnetic Material Science, Statistical Regression

---

## 4. Problem Statement
Traditional RUL models for appliances are limited by:
*   **Sensor Costs:** Requiring expensive vibration or flux sensors.
*   **Lagging Indicators:** Detecting faults (e.g., shorts) only after irreversible damage occurs.
*   **Material Blindness:** Failing to account for "silent" aging, such as the gradual decay of **magnetic permeability ($\mu$)** in the core or thinning of wire enamel.

---

## 5. The Multi-Physics Solution
The system creates a **Health Index (HI)** using a "Triple-Domain" approach, all extracted via the motor's own control firmware.

### A. The Signal Injection Method (The "Probing" Step)
The inverter modulates the PWM duty cycle to perform a frequency sweep (e.g., 1kHz - 20kHz) at standstill.
*   **Input:** Commanded Voltage Vector ($V^*$).
*   **Output:** Measured Current Response ($I_{meas}$).
*   **Calculation:** $Z(f) = V^*(f) / I_{meas}(f)$.

### B. Multi-Physics Indicators

| Category | Parameter | Physical Indicator of Aging |
| :--- | :--- | :--- |
| **Electrical** | $R_s$, Inductance ($L$) | Ohmic heating and winding shorts. |
| **Magnetic** | Flux Linkage ($\lambda$) | Permanent magnet weakening / Coercivity decay. |
| **Derived** | **Resonance Shift ($\Delta f_r$)** | **Core material aging** (Permeability decay). |
| **Derived** | **Peak Flattening ($Q$)** | **Insulation aging** (Dielectric loss). |

---

## 6. Novelty and Claims
### Key Novelty:
The use of the **resonance frequency peak** as a proxy for the physical state of the magnetic core material. As the core "fatigues," its ability to conduct magnetic flux (permeability) changes, causing the electrical resonance of the entire motor to shift in a mathematically predictable way ($f_r \approx 1/\sqrt{LC}$).

### Specific Claims:
1.  **Claim 1:** A method for extracting a motor impedance spectrum using **PWM carrier injection** to identify shifts in resonance frequency.
2.  **Claim 2:** A statistical model that correlates the **rate of resonance frequency shift ($\Delta f_r / \Delta t$)** with the degradation of magnetic core permeability.
3.  **Claim 3:** A "hardware-free" diagnostic framework that utilizes existing inverter phase-current sensors to calculate a **Multi-Physics Health Index**.

---

## 7. Statistical RUL Calculation
The RUL is estimated using a **Particle Filter** or **Exponential Regression** based on the Health Index (HI):
$$HI(t) = w_1 \cdot \Delta f_r + w_2 \cdot \Delta R + w_3 \cdot \Delta \lambda$$
Where $w_i$ are sensitivity weights. The RUL is defined as the time $t$ remaining until $HI(t)$ reaches a critical failure threshold defined by material safety limits.

## 8. Implementation in Field-Oriented Control (FOC) Systems
The proposed RUL model is uniquely compatible with Vector Motor Drives (FOC) by utilizing the decoupled control of flux and torque:

*   **D-Axis Perturbation:** High-frequency voltage is injected into the d-axis ($V_d$) to measure the **impedance resonance** of the magnetic circuit without impacting torque ripple or mechanical stability.
*   **Real-Time Parameter Estimation:** The FOC current regulators act as a natural filter, allowing the system to extract winding resistance and inductance changes as "model errors" during standard operation.
*   **Sensorless Symmetry:** Because FOC already requires high-resolution phase current sensing and rotor position estimation, no additional hardware is required to capture the **$\Delta f_r$ shift**.

## Block Diagram and System Architecture
graph LR
    subgraph "Standard FOC Loop"
    Ref[Speed/Torque Ref] --> PI[PI Current Controllers]
    PI --> InvPark[Inverse Park Transform]
    InvPark --> PWM[SVPWM / Inverter]
    PWM --> Motor((Motor))
    Motor --> ADC[Current Sensing]
    ADC --> Park[Park Transform]
    Park --> PI
    end

    subgraph "Health Monitor System"
    HM[Health Monitor / RUL Predictor]
    Inj[HF Injection Module]
    Ext[Signal Extraction & FFT]
    end

    %% Connections
    Inj -.->|Inject HF Signal| InvPark
    ADC -.->|Raw Current Data| Ext
    Ext -->|Impedance & Resonance| HM
    PI -->|Observe Control Effort| HM
    HM -->|RUL & Health Status| UI[User Interface / Cloud]
