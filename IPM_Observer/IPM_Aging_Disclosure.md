# IPM_Aging_Disclosure.md

## 1. Title of the Invention
**Hybrid Multi-Domain Prognostic Observer for Integrated Power Modules (IPM) using Active Protection-Loop Probing and ADC-based Thermal Transient Analysis.**

---

## 2. Prior Art and Novelty Analysis

### 2.1 Existing Patent Landscape
*   **Passive Monitoring (e.g., US20240402772A1):** Existing solutions focus on monitoring absolute voltage or temperature levels. They lack the ability to actively probe the internal logic state of the IPM.
*   **Gate Driver IC Protection (e.g., US11397209B2):** Hardware-based solutions measure delays within the gate driver chip but do not correlate these with thermal impedance or provide a statistical RUL via software.

### 2.2 Novelty of the Invention
1.  **Active Protection-Loop Probing:** Utilizing a dedicated Gate Driver Enable (GDU_EN) line to stimulate the **ITRIP** pin and measure the response latency on the **VFO** pin. This allows for driver-health diagnostics even when the motor is stationary.
2.  **Cross-Modal Verification:** Using ADC-derived thermal impedance data to validate whether timing drifts in the power stage (Shunt-based) or the logic stage (ITRIP-based) are permanent structural aging or temporary temperature fluctuations.
3.  **Triple-Domain Diagnostics:** A single software observer that independently identifies **Mechanical** (Solder), **Silicon** (IGBT Gate), and **Logic** (Driver) aging.

---

## 3. Technical Field
The present invention relates to power electronics and motor control systems, specifically to a non-intrusive, software-defined method for monitoring the health and estimating the Remaining Useful Life (RUL) of Integrated Power Modules (IPM).

---

## 4. Background of the Invention
Standard IPMs, such as the IKCM15L60GD, provide an ITRIP pin for over-current protection and a VFO pin for fault reporting and NTC-based temperature monitoring. Conventionally, these are used only for reactive safety shutdowns. This invention introduces an active diagnostic observer that utilizes these existing pins to measure sub-threshold degradation in the power, thermal, and logic domains.

---

## 5. Detailed Description of the Invention

### 5.1 System Overview
The invention is a firmware-defined "Aging Observer" utilizing three diagnostic paths:
1.  **ADC-based VFO/NTC Monitoring:** Decodes internal junction temperature transients.
2.  **Shunt-based Latency Mapping:** Measures switching speed via the emitter shunts.
3.  **Active ITRIP Probing:** Measures the propagation delay of the internal protection logic.

### 5.2 Thermal Transient Observer (ADC Path)
The MCU monitors the analog voltage on the VFO/NTC pin.
*   **Algorithm:** During a load transient, the MCU calculates the **Normalized ADC Slope** ($dV/dt / I^2$).
*   **Significance:** A steeper slope indicates increased internal thermal resistance ($R_{th}$), signaling solder fatigue or delamination.

### 5.3 Electrical Latency Observer (Shunt/Comparator Path)
The MCU utilizes an internal analog comparator to detect the onset of current at the NW, NV, or NU shunt terminals.
*   **Algorithm:** The system timestamps the interval between the PWM "ON" command and the comparator trip.
*   **Significance:** Drift in this latency (normalized for DC-link voltage and temperature) identifies IGBT gate-oxide wear.

### 5.4 Active Logic Observer (ITRIP Path)
During system initialization or idle states, the MCU executes a **"Health Check Pulse."**
*   **Algorithm:** The MCU asserts the GDU_EN line (connected to ITRIP) and measures the nanosecond-level delay until the VFO pin pulls LOW.
*   **Significance:** This measures the propagation delay of the internal logic and level-shifters. A drift here indicates aging of the internal driver circuitry or supply decoupling capacitors.

### 5.5 Statistical RUL Fusion
The system fuses these inputs into a **Bayesian Statistical Model** to predict the time remaining until the IPM violates safety-critical reliability limits.

---

## 6. List of Claims

**Claim 1:** A method for estimating the health of an IPM in a motor control system, comprising:
*   Sampling an analog voltage via an **ADC channel** to extract a thermal transient feature;
*   Capturing a switching latency via a **shunt-based comparator** to extract a power-stage timing feature;
*   Measuring a protection-loop latency via an **active ITRIP stimulus** to extract a logic-stage timing feature;
*   Calculating a combined Health Index (HI) from these features.

**Claim 2:** The method of Claim 1, wherein the thermal transient feature is a **Normalized ADC Slope** ($dV/dt / I^2$) representing internal thermal impedance shifts.

**Claim 3:** The method of Claim 1, wherein the power-stage timing feature is the temporal delta between a PWM transition and a shunt current rise, normalized against absolute temperature readings from the ADC.

**Claim 4:** The method of Claim 1, further comprising an **Active Diagnostic Mode** wherein the MCU initiates a pulse on the ITRIP pin and measures the response time on the VFO pin to isolate driver-logic aging.

**Claim 5:** A software-defined prognostic system that differentiates between package-level aging, silicon-level aging, and logic-level aging without additional external sensors.

**Claim 6:** A statistical RUL estimation algorithm that extrapolates the combined Health Index drift toward a predefined failure boundary.

---

## 7. Advantages
*   **Active Self-Test:** Can verify the health of the safety system even when the motor is idle.
*   **Zero-Budget:** Optimized for standard IKCM15L60GD architectures with no additional BOM cost.
*   **High Integrity:** Uses three independent physics domains to provide a high-confidence RUL estimate.
