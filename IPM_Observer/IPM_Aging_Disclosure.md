## 1. Title of the Invention
**Hybrid Multi-Domain Prognostic Observer for Integrated Power Modules (IPM) using Statistical Latency and Thermal Transient Analysis.**

---

## 2. Prior Art and Novelty Analysis

### 2.1 Existing Patent Landscape
A search of the current state of the art reveals several related but distinct methodologies:
*   **Case-Temperature Monitoring (e.g., US20240402772A1):** Utilizes external thermal sensors to measure junction temperature ($T_j$) and predict thermal behavior. These require **additional physical hardware** (external NTCs).
*   **Hardware-Integrated Detectors (e.g., US8736314B2):** Describes dedicated on-chip aging detectors. These require **semiconductor-level modifications** during manufacturing.
*   **Signal Injection Methods:** Use small-signal manipulation of gate-source voltage to measure phase delays. These are often **intrusive**, potentially interfering with standard motor control loops.

### 2.2 Novelty of the Invention
This invention distinguishes itself through three unique technical leaps:
1.  **Passive Signal Repurposing:** Unlike systems requiring new sensors, this invention repurposes the **VFO (Fault Output)** and **Gate-Sense** signals—lines already present in standard IPM architectures.
2.  **Cross-Modal Verification:** It is uniquely capable of differentiating between **package-level aging** (via $dT/dt$ slope analysis) and **silicon-level aging** (via gate-sense latency) in a single unified software observer.
3.  **Zero-Budget/Firmware-Only:** It requires no additional Bill-of-Materials (BOM) cost and no intrusive signal injection, allowing for backward compatibility via firmware updates.

---

## 3. Technical Field
The present invention relates to power electronics and motor control systems, specifically to a non-intrusive, software-defined method for monitoring the health and estimating the Remaining Useful Life (RUL) of Integrated Power Modules (IPM) within an inverter.

---

## 4. Background of the Invention
Conventional IPMs include reactive protection features such as Over-Current Protection (OCP) and Over-Temperature Protection (OTP). These only trigger after a failure has occurred. There is a critical need for a proactive solution that detects sub-threshold aging using only the signals already available to the motor control microcontroller (MCU).

---

## 5. Detailed Description of the Invention

### 5.1 System Overview
The invention is a firmware-defined "Aging Observer" that utilizes two existing feedback paths:
1.  **VFO (Fault Output):** Decodes internal junction temperature transients.
2.  **Gate-Sense (Feedback):** Measures nanosecond-level switching propagation delays.

### 5.2 Thermal Transient Observer (VFO Path)
The MCU calculates the **Normalized Thermal Impulse Response**. As solder layers fatigue, the thermal resistance ($R_{th}$) increases, leading to a steeper temperature rise ($dT/dt$) for the same current magnitude ($I^2$).
- **Algorithm:** $HI_{thermal} = (dT/dt)_{observed} / I_{phase}^2$
- **Significance:** Detects delamination and solder voids.

### 5.3 Electrical Latency Observer (Gate-Sense Path)
The MCU uses internal high-resolution timers to timestamp the interval ($t_{pd}$) between the PWM command and the Gate-Sense confirmation. This identifies shifts in the **Threshold Voltage ($V_{th}$)** and Miller Plateau duration.
- **Algorithm:** $\Delta t = Capture\_Time_{Sense} - Capture\_Time_{PWM}$
- **Significance:** Detects gate-oxide degradation and semiconductor wear-out.

### 5.4 Statistical RUL Fusion
The system implements a **Bayesian Update Model** to predict failure:
1.  **Digital Birth Certificate:** Records "Golden Signatures" during the first 100 hours of operation.
2.  **Drift Analysis:** Tracks the moving average of both signals.
3.  **Cross-Validation:** If a timing shift is detected without a thermal shift, it is flagged as silicon wear. If both shift, it indicates accelerated end-of-life.

---

## 6. List of Claims

**Claim 1:** A method for estimating the health of an Integrated Power Module (IPM) in a motor control system, the method comprising:
*   Capturing a thermal feedback signal (VFO) and an electrical feedback signal (Gate-Sense) using existing microcontroller peripherals;
*   Extracting a thermal transient feature and an electrical timing feature from said signals;
*   Calculating a combined Health Index (HI) based on the divergence of these features from a stored baseline.

**Claim 2:** The method of Claim 1, wherein the thermal transient feature is a normalized slope ($dT/dt / I^2$) used as a proxy for the thermal resistance ($R_{th}$) of the module’s packaging.

**Claim 3:** The method of Claim 1, wherein the electrical timing feature is a propagation delay measurement ($t_{pd}$) between a PWM command and a gate-sense feedback signal to identify shifts in gate-charge dynamics.

**Claim 4:** The method of Claim 3, wherein the electrical timing feature further comprises a differential analysis between turn-on and turn-off latencies to isolate gate threshold voltage ($V_{th}$) shifts from driver-circuit drift.

**Claim 5:** A software-defined prognostic system that differentiates between package-level aging and silicon-level aging by cross-referencing the thermal transient feature of Claim 2 with the electrical timing feature of Claim 3.

**Claim 6:** A statistical Remaining Useful Life (RUL) estimation algorithm that calculates the rate of change of the combined Health Index and extrapolates the time remaining until a safety-critical degradation threshold is reached.

---

## 7. Advantages
*   **No Hardware Cost:** Works on existing hardware architectures.
*   **Proactive:** Detects failures before they cause system downtime.
*   **Robust:** Cross-modal analysis (Thermal + Electrical) prevents false alarms from ambient temperature changes.
