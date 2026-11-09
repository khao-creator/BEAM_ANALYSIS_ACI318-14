# BEAM ANALYSIS ACI 318-14

[![Python Version](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org)
[![Validation](https://img.shields.io/badge/validation-3%20cases%20audited-green.svg)](#validation--benchmarks)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

A high-precision Python desktop application for the structural verification and safety audit of reinforced concrete beam sections under flexure, shear, and combined torsion according to ACI 318-14 specifications.

![GUI Screenshot](assets/Screenshot.png)

---

## Features & Engine Pipeline

An integrated cross-section verification panel executing step-by-step structural checks across five engineering modules:

```mermaid
graph LR
    M1["<b>MODULE 1</b><br/>Geometry & Materials"] --> M2["<b>MODULE 2</b><br/>Flexure Engine"]
    M2 --> M3["<b>MODULE 3</b><br/>Shear Engine"]
    M3 --> M4["<b>MODULE 4</b><br/>Torsion Engine"]
    M4 --> M5["<b>MODULE 5</b><br/>Anchorage & Detailing"]

    style M1 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M2 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M3 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M4 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M5 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
```

* **Module 1 – Ingestion & Geometry:** Ingests metric MKS units, builds rebar layer depth centroids, and maps effective depths ($d, d'$).
* **Module 2 – Strain Compatibility Flexure:** Bisection equilibrium solver evaluating strain limits, tension-controlled $\phi$ mapping ($0.65 - 0.90$), and axial load limit ($P_u \le 0.10 f'_c A_g$).
* **Module 3 – Shear Capacity:** Evaluates $V_c$ with axial compression/tension effects, minimum stirrup area, stirrup spacing limits ($s_{\text{max}}$), and web crushing capacity.
* **Module 4 – Combined Torsion Interaction:** Evaluates threshold $\phi T_{\text{th}}$, cracking $\phi T_{\text{cr}}$, closed stirrup ratios ($A_t/s$), longitudinal torsion steel ($A_l$), and combined shear-torsion web stress limits.
* **Module 5 – Anchorage & Detailing:** Computes tension development lengths ($L_d$) according to ACI 25.4.2 based on clear bar spacing and confinement.

---

## Quick Start

### Requirements
- **Python:** 3.9 or higher
- **Dependencies:** Listed in `requirements.txt`

### Installation & Execution

1. Clone the repository and install dependencies:
   ```bash
   git clone https://github.com/your-username/beam-analysis-aci318.git
   cd beam-analysis-aci318
   pip install -r requirements.txt
   ```

2. Run the application:
   ```bash
   python "concrete beam reinforcement ACI 318-14.py"
   ```

---

## Validation & Benchmarks

The program was verified against three independent hand calculations performed per ACI 318-14. Case 1 checks baseline behavior under low torsion. Case 2 deliberately stacks complex branch conditions (two-layer reinforcement, high-strength concrete, axial tension/compression, full torsion interaction) to audit formulation edge cases. Case 3 is intentionally non-compliant to confirm that the program never reports an unsafe section as SAFE. All technical deviations are detailed in [Known Deviations & Engineering Assumptions](#known-deviations--engineering-assumptions).

| Case | Question Answered | What It Exercises | Result Summary | Hand Calculation |
| :--- | :--- | :--- | :--- | :---: |
| **Case 1: Base** | Is the program accurate under normal design conditions? | Singly and doubly reinforced sections, $\phi = 0.90$, $T_u < \phi T_{\text{th}}$ (torsion correctly neglected). | $\phi M_n$ and $\phi V_n$ differ by $< 0.1\%$ vs hand calculations. | [PDF](./validation/case1_hand_calc.pdf) |
| **Case 2: Edge** | How does the engine perform when complex conditions stack? | 2-layer steel layout, high-strength concrete ($f'_c = 350$ ksc), axial tension/compression, full torsion design. | Documented deviations observed ($\phi / \phi M_n \approx 1.9\%$ with $\epsilon_t \approx 7.2\%$, $s_{\max} \approx +100\%$, $T_{\text{th}} \approx \pm 35\%$, $A_l \approx +112\%$). Section END correctly flagged UNSAFE ($UR = 1.097$). | [PDF](./validation/case2_hand_calc.pdf) |
| **Case 3: Fail** | Does the program correctly reject unsafe or non-compliant sections? | Tight rebar spacing violation, $\epsilon_t < 0.004$, $A_s < A_{s,\min}$, $P_u$ cap exceedance, $s > s_{\max}$, $M_u > \phi M_n$, $V_u > \phi V_n$. | Successfully rejects non-compliant inputs ($3/3$ sections flagged as UNSAFE). | [PDF](./validation/case3_hand_calc.pdf) |

---

## Known Deviations & Engineering Assumptions

During benchmark validation against hand calculations, the following code implementation choices and analytical assumptions were documented:

1. **Uncoupled Torsional Threshold ($\phi T_{\text{th}}$ vs Axial Load $P_u$):** The program evaluates uncoupled pure torsion threshold strength ($\phi T_{\text{th}}$) without the axial modifier term $\sqrt{1 + \frac{P_u}{1.06 A_g \sqrt{f'_c}}}$ in MKS units (where $P_u > 0$ for compression and $P_u < 0$ for tension, per ACI 318-14 Sec 22.7.4.1). Under net axial tension, this yields a higher, unconservative threshold (+35% at INITIAL section in Case 2). *Refactoring to couple $P_u$ in $T_{\text{th}}$ is scheduled for the next patch.*
2. **Tensile Strain Depth Baseline ($d$ vs $d_t$):** Net tensile strain $\epsilon_t$ is evaluated at the centroid depth of the tension steel group ($d$) rather than the extreme tension bar layer depth ($d_t$). This is a deliberate conservative deviation from ACI 21.2.2, resulting in lower-bound $\phi$ factor mapping on multi-layer beams.
3. **Concrete Displacement by Compression Steel ($d' < c$):** Concrete area displaced by compression steel is subtracted whenever $d' < c$ ($C_c = 0.85 f'_c a b - 0.85 f'_c A'_s$). This introduces a minor 1.1%–1.7% shift in neutral axis depth $c$ compared to hand calculations that check $d' < a$.
4. **Stirrup Spacing Limit Threshold Baseline ($s_{\max}$):** The stirrup spacing threshold comparator ($1.1 \sqrt{f'_c} b d$) in Table 9.7.6.2.2 is evaluated against required stirrup demand ($V_{s,\text{req}}$) rather than provided capacity ($V_{s,\text{prov}}$). In Case 2, this permitted $s_{\max} = d/2$ instead of $d/4$ ($\approx +100\%$ higher spacing limit), which is less conservative than checking $V_{s,\text{prov}}$.
5. **Longitudinal Torsion Steel ($A_l$) Credit:** Excess flexural tension and compression reinforcement ($A_{s,\text{prov}} - A_{s,\text{req}}$) are credited toward longitudinal torsion steel capacity per ACI 318-14 Sec 9.5.4.3, 9.5.4.4, and Sec 22.7.6.2.2 (+112% capacity credit compared to web-bar-only accounting).

---

## Theory & Mathematical Formulations

*(Note: $1.06 \sqrt{f'_c}$ is used in the $T_{\text{th}}$ axial modifier term for exact MKS conversion from $0.33 \sqrt{f'_c} \text{ MPa}$; $1.1 \sqrt{f'_c}$ is used elsewhere for consistency with common Thai MKS engineering practice.)*

<details>
<summary><b>Click to expand Module 2: Flexure Engine Formulations</b></summary>

### Flexure Formulations (ACI 318-14 Chapter 22)

* **Whitney Stress Block Depth:**
  $$a = \beta_1 c$$

* **Concrete Compression Force:**
  $$C_c = 0.85 f'_c a b$$

* **Steel Stress (bounded by yield limits):**
  $$f_{si} = \max\left(-f_y, \, \min(f_y, \, E_s \epsilon_{si})\right)$$

* **Section Force Equilibrium:**
  $$C_c + \sum C_s - \sum T_s - P_u = 0$$

* **Strength Reduction Factor Mapping ($\phi$):**

$$\phi = \begin{cases} 0.90 & \text{if } \epsilon_t \ge 0.005 \text{ (Tension-controlled)} \\ 0.65 + 0.25 \frac{\epsilon_t - \epsilon_{ty}}{0.005 - \epsilon_{ty}} & \text{if } \epsilon_{ty} < \epsilon_t < 0.005 \text{ (Transition)} \\ 0.65 & \text{if } \epsilon_t \le \epsilon_{ty} \text{ (Compression-controlled)} \end{cases}$$

* **Nominal Moment Capacity (about centroid $y_c = h/2$):**
  $$M_n = \sum F_i (y_i - y_c)$$

* **Design Moment Verification:**
  $$\phi M_n \ge M_u$$

* **Beam Module Axial Limit (ACI 9.3.1.1):**
  $$P_u \le 0.10 f'_c A_g$$

</details>

<details>
<summary><b>Click to expand Module 3: Shear Engine Formulations</b></summary>

### Shear Formulations (ACI 318-14 Chapter 22)

* **Concrete Shear Strength $V_c$ ($\phi = 0.75$ applied to $V_n$ per ACI Table 21.2.1):**
  * **Axial Compression ($P_u > 0$):**
    $$V_c = 0.53 \left(1 + \frac{P_u}{140 A_g}\right) \sqrt{f'_c} b d$$
  * **Axial Tension ($P_u < 0$):**
    $$V_c = \max\left(0, \, 0.53 \left(1 + \frac{P_u}{35 A_g}\right) \sqrt{f'_c} b d\right)$$
  * **Zero Axial Load ($P_u = 0$):**
    $$V_c = 0.53 \sqrt{f'_c} b d$$

* **Stirrup Shear Strength Contribution ($V_s$):**
  $$V_s = \frac{A_v f_{yt} d}{s}$$

* **Design Shear Strength Verification:**
  $$\phi V_n = \phi (V_c + V_s) \ge V_u \quad (\text{with } \phi = 0.75)$$

* **Stirrup Spacing Limits ($s_{\text{max}}$):**

$$s_{\text{max}} = \begin{cases} \min(d/4, \, 30 \text{ cm}) & \text{if } V_{s,\text{req}} > 1.1 \sqrt{f'_c} b d \\ \min(d/2, \, 60 \text{ cm}) & \text{if } V_{s,\text{req}} \le 1.1 \sqrt{f'_c} b d \end{cases}$$

* **Minimum Shear Reinforcement Ratio:**
  $$\left(\frac{A_v}{s}\right)_{\text{min}} = \max\left(\frac{0.2 \sqrt{f'_c} b}{f_{yt}}, \, \frac{3.5 b}{f_{yt}}\right)$$

* **Maximum Stirrup Shear Capacity Limit:**
  $$V_s \le 2.1 \sqrt{f'_c} b d$$

</details>

<details>
<summary><b>Click to expand Module 4: Combined Torsion Formulations</b></summary>

### Combined Torsion Formulations (ACI 318-14 Chapter 22)

*(Note: Torsion equations assume a space truss angle of $\theta = 45^\circ$, giving $\cot \theta = 1.0$.)*

* **Torsion Geometry (Stirrup Centerline):**
  $$x_1 = b - 2(\text{cover} + d_{\text{stirrup}}/2), \quad y_1 = h - 2(\text{cover} + d_{\text{stirrup}}/2)$$
  $$A_{oh} = x_1 \cdot y_1, \quad p_h = 2(x_1 + y_1), \quad A_o = 0.85 A_{oh}$$

* **Design Torsional Threshold and Cracking Strengths ($\phi = 0.75$):**
  $$\phi T_{\text{th}} = \phi \cdot 0.27 \sqrt{f'_c} \left(\frac{A_{\text{cp}}^2}{P_{\text{cp}}}\right), \quad \phi T_{\text{cr}} = \phi \cdot 1.1 \sqrt{f'_c} \left(\frac{A_{\text{cp}}^2}{P_{\text{cp}}}\right)$$

* **Combined Shear-Torsion Web Crushing Limit ($\phi = 0.75$):**
  $$\sqrt{\left(\frac{V_u}{b d}\right)^2 + \left(\frac{T_u p_h}{1.7 A_{oh}^2}\right)^2} \le \phi \left(\frac{V_c}{b d} + 2.1\sqrt{f'_c}\right)$$

* **Transverse Torsion Reinforcement (per single leg):**
  $$\frac{A_t}{s} = \frac{T_u}{2 \phi A_o f_{yt}} \quad (\phi = 0.75)$$

* **Governing Transverse Stirrup Ratio:**
  $$\text{Governing ratio} = \max\left(\frac{A_{v,\text{shear}}}{s} + \frac{2 A_t}{s}, \, \left(\frac{A_v}{s}\right)_{\text{min}}\right)$$

* **Maximum Torsion Stirrup Spacing Limit:**
  $$s_{\text{max}} = \min(p_h / 8, \, 30 \text{ cm})$$

* **Longitudinal Torsion Reinforcement ($A_l$):**
  $$A_l = \max\left(A_{l,\text{req}}, \, \max(0.0, \, A_{l,\text{min}})\right)$$
  $$A_{l,\text{req}} = \left(\frac{A_t}{s}\right) p_h \left(\frac{f_{yt}}{f_y}\right)$$
  $$A_{l,\text{min}} = \frac{1.33 \sqrt{f'_c} A_{\text{cp}}}{f_y} - \left(\frac{A_t}{s}\right)_{\text{clamped}} p_h \left(\frac{f_{yt}}{f_y}\right)$$
  $$\text{where } \left(\frac{A_t}{s}\right)_{\text{clamped}} = \max\left(\frac{A_t}{s}, \, \frac{1.78 b}{f_{yt}}\right)$$

</details>

---

## Workflow & Data Example

The application handles configurations via an `input.json` file located in the root directory. You can edit variables via the file directly or use the interactive Tkinter entry bars.

```jsonc
{
  "fc_prime": 240,       // Concrete compressive strength (kg/cm²)
  "fy": 4000,            // Longitudinal reinforcement yield strength (kg/cm²)
  "fy_shear": 4000,      // Stirrup yield strength (kg/cm²)
  "b": 30, "h": 60,      // Cross section width and height (cm)
  "covering": 6,         // Concrete cover to stirrup outer edge (cm)
  "stirrup_dia": 9,      // Stirrup bar diameter (mm)
  "stirrup_spacing": 20, // Stirrup spacing (cm)
  "stirrup_legs": 2,     // Number of shear legs
  "forces": {
    "INITIAL": { "Mu": 45000, "Vu": 8000, "Pu": 0, "Tu": 0 },
    "MID":     { "Mu": 45000, "Vu": 8000, "Pu": 0, "Tu": 0 },
    "END":     { "Mu": 0,     "Vu": 0,    "Pu": 0, "Tu": 0 }
  },
  "rebars": {
    "INITIAL": { "top": [{"dia": 20, "qty": 2, "clear_dist": 0}], "bot": [{"dia": 25, "qty": 4, "clear_dist": 0}] },
    "MID":     { "top": [{"dia": 20, "qty": 2, "clear_dist": 0}], "bot": [{"dia": 25, "qty": 4, "clear_dist": 0}] },
    "END":     { "top": [{"dia": 20, "qty": 2, "clear_dist": 0}], "bot": [{"dia": 20, "qty": 2, "clear_dist": 0}] }
  }
}
```

---

## Engineering Boundaries & Limitations

- **Cross-Section Geometry:** Bounded strictly to solid rectangular sections. T-beams, L-beams, or hollow box sections are excluded.
- **Seismic Detailing Scope:** Calculations assume ordinary moment frame members. Special Moment Frame (SMF) seismic hoop spacing and capacity design provisions are out of scope.
- **Limit States:** Applies strictly to Ultimate Strength Limit States (ULS). Serviceability checks (immediate/long-term deflections, crack width limits) must be performed independently.

---

## Author

**ARIYA**  
Lead Structural Engineer & Developer of BEAM ANALYSIS ACI 318-14.

---

## Credits & Acknowledgments

### Core Infrastructure & Numerical Engine
Developed by **ARIYA**:
- Bisection numerical equilibrium solver mechanics.
- Metric MKS unit ingestion pipeline.
- Tkinter UI components and visual canvas rendering.

### Development & AI Co-Piloting
- **Engineering Architecture & Validation:** Conceptualized, formulated, and engineered by **ARIYA**. All mathematical formulations, numerical solvers, and verification checks were independently verified against hand calculations.
- **Code Implementation:** Artificial intelligence tools were utilized as a co-pilot to assist in Python code implementation, UI canvas prototyping, and documentation formatting under strict engineering supervision.

### Academic & Compliance References
- **ACI 318-14 / ACI 318M-14:** *Building Code Requirements for Structural Concrete*, American Concrete Institute.
- **Prof. Monchai (อาจารย์มนชัย):** Structural Engineering lecture notes and calculation methodologies for reinforced concrete design (ACI Ultimate Strength Design - USD method).

### Third-Party References
- **ConcreteDesignPy:** The underlying workflow sequence for shear/torsion interaction logic and axial-load conditional logic was built referencing the open-source architecture of [ConcreteDesignPy](https://github.com/albertp16/concretedesignpy) by Albert Pamonag Engineering Consultancy, used under the terms of the MIT License. See [`THIRD-PARTY-NOTICES.md`](THIRD-PARTY-NOTICES.md) for details.

---

## Disclaimer

This software is provided "as is" for educational, research, and preliminary design verification purposes. Calculations are benchmarked strictly against independent hand-calculations and have not been cross-verified against commercial design software. All analytical outputs must be reviewed, verified, and stamped by a licensed professional structural engineer before implementation in physical construction projects. The author assumes no liability for structural decisions or financial consequences resulting from the use of this software.

---

## License

Distributed under the MIT License. See [`LICENSE`](LICENSE) for details.