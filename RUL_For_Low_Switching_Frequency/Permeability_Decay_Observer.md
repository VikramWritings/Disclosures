# Technical Specification: Motor Core Permeability Decay Observer
## Target Platform: Renesas RA6T3 (Cortex-M33)

This document describes the implementation of a firmware-based observer to detect magnetic permeability decay ($\mu$) in a washer motor. This is achieved by tracking **d-axis inductance ($L_d$)** using High-Frequency Signal Injection (HFI).

---

## 1. System Overview
Since permeability decay occurs over thousands of operating hours, the firmware compares current inductance measurements against a "Birth Record" stored in non-volatile memory.


| Component             | Specification                           |
| :-------------------- | :-------------------------------------- |
| **MCU**               | Renesas RA6T3                           |
| **Control Frequency** | 16 kHz (PWM & FOC Loop)                 |
| **Injection Type**    | Sinusoidal High-Frequency (d-axis)      |
| **Injection Freq**    | 2 kHz                                   |
| **Thermal Sensing**   | Sensorless (via Stator Resistance $R_s$)|

---

## 2. Mathematical Logic
The motor's inductance is defined by:
$$L = \frac{N^2 \cdot \mu \cdot A}{l}$$
Where $\mu$ is the permeability. To measure $\mu$ decay, we isolate $L$ by injecting a high-frequency voltage ($V_h$) and measuring the current response ($I_h$):
$$L_d \approx \frac{V_h}{\omega_h \cdot I_h}$$

To account for the lack of a temperature sensor, the algorithm uses a **Cold-State Validation** gate. Data is only saved when the estimated stator resistance ($R_s$) matches the factory ambient value, ensuring $L$ changes are due to material decay, not heat.

---

## 3. Firmware Architecture

### A. Peripheral Configuration (RA6T3)
*   **GPT (Timer)**: Configured for Center-Aligned PWM at 16kHz. GPT triggers the ADC at the carrier peaks.
*   **ADC12**: Samples phase currents. Using the RA6T3's **Double Data Register**, we capture current at $t=0$ and $t=T/2$ to filter out PWM noise.
*   **TFU (Trig Unit)**: Used to generate the $sin$ wave for the 2kHz injection signal.

### B. Signal Processing Flow
1.  **Inject**: Add a 2kHz sine wave to the $d$-axis voltage reference ($V_{d\_ref}$).
2.  **Filter**: Apply a Digital Band-Pass Filter (BPF) to the measured $I_d$ to extract only the 2kHz component ($I_{h}$).
3.  **Calculate**: Solve for $L_d$ using the RMS values of $V_h$ and $I_h$.
4.  **Trend**: Store the result in **Data Flash** once every 50 cycles.

---

## 4. Implementation Example (C-Code)

```c
/*
 * RA6T3 Permeability Decay Observer
 * Frequency: 16kHz PWM / 2kHz Injection
 */

#include "hal_data.h"

#define INJECTION_FREQ_HZ 2000.0f
#define OMEGA_H           (2.0f * 3.14159f * INJECTION_FREQ_HZ)
#define DECAY_THRESHOLD   0.90f  // 10% drop indicates significant aging

typedef struct {
    float l_d_baseline;    // Factory value stored in Flash
    float l_d_current;     // Moving average of current measurement
    float rs_estimated;    // Virtual temperature proxy
} MotorHealth_t;

void run_health_observer(float v_d_inj, float i_d_measured) {
    // 1. Extract high-frequency current using a BPF (Simplified)
    static float i_h_prev = 0.0f;
    float i_h = apply_bpf_2khz(i_d_measured);

    // 2. Calculate Instantaneous Inductance
    // L = V / (omega * I)
    if (i_h > 0.001f) { // Prevent division by zero
        float l_inst = v_d_inj / (OMEGA_H * i_h);

        // 3. Update Moving Average Filter
        static float l_filt = 0.0f;
        l_filt = (0.999f * l_filt) + (0.001f * l_inst);

        // 4. Check for logging conditions (Cold Motor)
        if (estimate_rs() < RS_AMBIENT_MAX) {
            log_to_flash(l_filt);
        }
    }
}
```

---

## 5. Decision Matrix (Fault Handling)


| Observed L-Drop | Status   | Firmware Action                          |
| :-------------- | :------- | :--------------------------------------- |
| < 5%            | Healthy  | Normal operation.                        |
| 5% - 10%        | Warning  | Store error code; suggest maintenance.   |
| > 10%           | Critical | Derate max torque to prevent saturation. |

---

## 6. VS Code Integration Tips
*   Use the **C/C++ Extension** for IntelliSense on RA6T3 registers.
*   Enable **Markdown Preview** (`Ctrl+Shift+V`) to view this spec formatted.
*   Use **Renesas Configurator** to ensure GPT and ADC triggers are hardware-synced to avoid software jitter in the 2kHz extraction.
