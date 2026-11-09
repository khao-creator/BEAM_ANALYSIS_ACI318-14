# BEAM ANALYSIS ACI 318-14

[![Python Version](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org)
[![Validation](https://img.shields.io/badge/validation-3%20cases%20audited-green.svg)](#validation--benchmarks)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

A high-precision Python desktop application for the structural verification and safety audit of reinforced concrete beam sections under flexure, shear, and combined torsion according to ACI 318-14 / ACI 318M-14 specifications in Metric MKS units ($\text{kg}$, $\text{cm}$, $\text{ksc}$, $\text{ton}$, $\text{t-m}$).

![GUI Screenshot](assets/Screenshot.png)

---

## Features & Visual Engine Pipeline

An integrated cross-section verification panel executing step-by-step structural checks across five modular calculation steps:

```mermaid
graph LR
    M1["<b>MODULE 1</b><br/>Geometry & Ingestion"] --> M2["<b>MODULE 2</b><br/>Flexure Engine"]
    M2 --> M3["<b>MODULE 3</b><br/>Shear Engine"]
    M3 --> M4["<b>MODULE 4</b><br/>Torsion Engine"]
    M4 --> M5["<b>MODULE 5</b><br/>Anchorage & Detailing"]

    style M1 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M2 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M3 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M4 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
    style M5 fill:#0f172a,stroke:#38bdf8,stroke-width:2px,color:#f8fafc
```

### Module Summary & Engineering Intuition

* **Module 1 – Ingestion & Layer Geometry:** Ingests MKS metric inputs ($b, h, f'_c, f_y, f_{yt}$), computes rebar layer centroids ($y_{\text{bar}}$), effective depths ($d, d'$), and verifies bar clearance limits.
* **Module 2 – Strain Compatibility Flexure:** Solves strain equilibrium using numerical bisection. Maps section ductility using net tensile strain ($\epsilon_t$) to determine strength reduction factor ($\phi = 0.65 - 0.90$).
  * *Intuition:* Ensures the beam gives ample warning through visible cracking ($\epsilon_t \ge 0.005$) rather than failing suddenly.
* **Module 3 – Shear Capacity Engine:** Evaluates concrete shear strength ($V_c$) with axial load modifiers ($P_u$), stirrup contribution ($V_s$), minimum shear steel, and web crushing capacity ($V_{s,\max}$).
  * *Intuition:* Concrete acts as the first line of defense ($V_c$). If shear demand exceeds $V_c$, closed stirrups ($V_s$) take over to prevent diagonal shear failure.
* **Module 4 – Combined Torsion Interaction:** Calculates threshold torsion ($\phi T_{\text{th}}$), cracking torsion ($\phi T_{\text{cr}}$), transverse closed stirrups ($A_t/s$), longitudinal torsion steel ($A_l$), and combined shear-torsion web stress limits.
  * *Intuition:* Torsion behaves like twisting a towel. If $T_u \le \phi T_{\text{th}}$, pure concrete skin is sufficient and torsion can be safely neglected. If $T_u > \phi T_{\text{th}}$, a full 3D steel cage (closed hoops + longitudinal bars) is required.
* **Module 5 – Anchorage & Detailing:** Computes tension development length ($L_d$) according to ACI 25.4.2 based on clear bar spacing, confinement, and bar size.

---

## Validation & Benchmarks

The calculation engine has been verified against three independent hand-calculated benchmark cases per ACI 318-14 to ensure complete reliability.

| Benchmark Case | Core Objective | Engineering Conditions Audited | Engine Accuracy vs Hand Calc | Calculation Proof Sheet |
| :--- | :--- | :--- | :--- | :---: |
| **Case 1: Base** | Normal Baseline Design | Singly & doubly reinforced sections, tension-controlled $\phi = 0.90$, $T_u \le \phi T_{\text{th}}$ (torsion safely neglected). | $\phi M_n$ and $\phi V_n$ differ by $< 0.1\%$. | [PDF Hand Calc](./validation/case1_hand_calc.pdf) |
| **Case 2: Edge** | Complex Stacked Edge Cases | 2-layer steel, high-strength concrete ($f'_c = 350\text{ ksc}$), axial tension & compression, full torsion interaction. | Documented minor formulation shifts ($\approx 1.9\%$ on $\phi M_n$). END section correctly flagged **UNSAFE** ($UR = 1.097$). | [PDF Hand Calc](./validation/case2_hand_calc.pdf) |
| **Case 3: Fail** | Unsafe Section Gatekeeping | Tight bar clearance violation, $\epsilon_t < 0.004$, $A_s < A_{s,\min}$, $P_u > 0.10 f'_c A_g$, $s > s_{\max}$, demand exceedance ($M_u > \phi M_n$). | 100% Detection Rate ($3/3$ non-compliant sections flagged **UNSAFE**). | [PDF Hand Calc](./validation/case3_hand_calc.pdf) |

---

## Known Deviations & Engineering Assumptions

During benchmark audits against hand calculations, five specific implementation choices and assumptions were documented:

1. **Uncoupled Torsional Threshold ($\phi T_{\text{th}}$ vs Axial Load $P_u$):**
   * *Implementation:* The engine evaluates pure torsional threshold ($\phi T_{\text{th}}$) uncoupled from axial load $P_u$. 
   * *Engineering Impact:* Under net axial tension ($P_u < 0$), this yields a slightly higher threshold (+35% at INITIAL section in Case 2) compared to including the axial factor $\sqrt{1 + \frac{P_u}{0.33 A_g \sqrt{f'_c}}}$.
2. **Tensile Strain Depth Baseline ($d$ vs $d_t$):**
   * *Implementation:* Net tensile strain $\epsilon_t$ is evaluated at the centroid depth of the tension steel group ($d$) rather than the extreme tension bar layer ($d_t$).
   * *Engineering Impact:* A conservative deviation from ACI 21.2.2 that maps lower-bound $\phi$ factors on multi-layer steel layouts.
3. **Concrete Displacement by Compression Steel ($d' < c$):**
   * *Implementation:* Concrete area displaced by compression steel is subtracted whenever $d' < c$ ($C_c = 0.85 f'_c a b - 0.85 f'_c A'_s$).
   * *Engineering Impact:* Introduces a minor 1.1%–1.7% neutral axis shift ($c$) relative to hand calculations that check $d' < a$.
4. **Stirrup Spacing Limit Threshold Baseline ($s_{\max}$):**
   * *Implementation:* The stirrup spacing limit comparator ($1.1 \sqrt{f'_c} b d$) in Table 9.7.6.2.2 is evaluated against required stirrup demand ($V_{s,\text{req}}$) rather than provided capacity ($V_{s,\text{prov}}$).
5. **Longitudinal Torsion Steel ($A_l$) Credit:**
   * *Implementation:* Excess flexural tension and compression reinforcement ($A_{s,\text{prov}} - A_{s,\text{req}}$) are credited toward longitudinal torsion steel demand per ACI 318-14 Sec 9.5.4.3 and Sec 22.7.6.2.2.

---

## Theory & Mathematical Formulations

*(Note: In Metric MKS units, $1.06 \sqrt{f'_c}$ corresponds to the exact SI conversion of $0.33 \sqrt{f'_c}\text{ MPa}$, while $1.10 \sqrt{f'_c}$ is commonly used for consistency with Thai engineering practice.)*

<details>
<summary><b>Click to expand Module 2: Flexure Engine Formulations</b></summary>

### Flexure Formulations (ACI 318-14 Chapter 22)

* **Whitney Stress Block Depth:**
<div align="center">

$$a = \beta_1 c$$

</div>

* **Concrete Compression Force:**
<div align="center">

$$C_c = 0.85 f'_c a b$$

</div>

* **Compression Steel Force (displaced concrete accounted for):**
<div align="center">

$$C'_s = A'_s (f'_s - 0.85 f'_c)$$

</div>

* **Section Force Equilibrium:**
<div align="center">

$$C_c + C'_s = T_s + P_u$$

</div>

* **Strength Reduction Factor Mapping ($\phi$):**
<div align="center">

$$\phi = \begin{cases} 0.90 & \text{if } \epsilon_t \ge 0.005 \quad \text{(Tension-controlled)} \\ 0.65 + 0.25 \frac{\epsilon_t - \epsilon_{ty}}{0.005 - \epsilon_{ty}} & \text{if } \epsilon_{ty} < \epsilon_t < 0.005 \quad \text{(Transition zone)} \\ 0.65 & \text{if } \epsilon_t \le \epsilon_{ty} \quad \text{(Compression-controlled)} \end{cases}$$

</div>

* **Nominal Moment Capacity (about section centroid $y_c = h/2$):**
<div align="center">

$$M_n = M_{Cc} + M_{Cs} + M_{Ts}$$

</div>

* **Design Moment Verification:**
<div align="center">

$$\phi M_n \ge M_u$$

</div>

* **Beam Axial Load Applicability Limit (ACI 9.3.1.1):**
<div align="center">

$$P_u \le 0.10 f'_c A_g$$

</div>

</details>

<details>
<summary><b>Click to expand Module 3: Shear Engine Formulations</b></summary>

### Shear Formulations (ACI 318-14 Chapter 22)

* **Concrete Shear Resistance ($V_c$ with $\phi = 0.75$):**
  * **Axial Compression ($P_u > 0$):**
<div align="center">

$$V_c = 0.53 \left(1 + \frac{P_u}{140 A_g}\right) \sqrt{f'_c} b d$$

</div>

  * **Axial Tension ($P_u < 0$):**
<div align="center">

$$V_c = \max\left(0, \, 0.53 \left(1 + \frac{P_u}{35 A_g}\right) \sqrt{f'_c} b d\right)$$

</div>

  * **Zero Axial Load ($P_u = 0$):**
<div align="center">

$$V_c = 0.53 \sqrt{f'_c} b d$$

</div>

* **Stirrup Shear Capacity ($V_s$):**
<div align="center">

$$V_s = \frac{A_v f_{yt} d}{s}$$

</div>

* **Design Shear Verification:**
<div align="center">

$$\phi V_n = \phi (V_c + V_s) \ge V_u \quad (\phi = 0.75)$$

</div>

* **Maximum Stirrup Spacing Limits ($s_{\max}$):**
<div align="center">

$$s_{\max} = \begin{cases} \min(d/4, \, 30 \text{ cm}) & \text{if } V_{s,\text{req}} > 1.1 \sqrt{f'_c} b d \\ \min(d/2, \, 60 \text{ cm}) & \text{if } V_{s,\text{req}} \le 1.1 \sqrt{f'_c} b d \end{cases}$$

</div>

* **Minimum Shear Reinforcement Ratio:**
<div align="center">

$$\left(\frac{A_v}{s}\right)_{\min} = \max\left(\frac{0.2 \sqrt{f'_c} b}{f_{yt}}, \, \frac{3.5 b}{f_{yt}}\right)$$

</div>

* **Concrete Web Crushing Limit:**
<div align="center">

$$V_s \le 2.1 \sqrt{f'_c} b d$$

</div>

</details>

<details>
<summary><b>Click to expand Module 4: Combined Torsion Formulations</b></summary>

### Combined Torsion Formulations (ACI 318-14 Chapter 22)

*(Note: Torsion equations assume a space truss angle of $\theta = 45^\circ$, giving $\cot \theta = 1.0$.)*

* **Torsion Core Geometry (Stirrup Centerline):**
<div align="center">

$$x_1 = b - 2(\text{cover} + d_{\text{stirrup}}/2), \quad y_1 = h - 2(\text{cover} + d_{\text{stirrup}}/2)$$

$$A_{oh} = x_1 \cdot y_1, \quad p_h = 2(x_1 + y_1), \quad A_o = 0.85 A_{oh}$$

</div>

* **Threshold and Cracking Torsion Capacities ($\phi = 0.75$):**
<div align="center">

$$\phi T_{\text{th}} = \phi \cdot 0.27 \sqrt{f'_c} \left(\frac{A_{\text{cp}}^2}{p_{\text{cp}}}\right) \sqrt{1 + \frac{P_u}{0.33 A_g \sqrt{f'_c}}}$$

$$\phi T_{\text{cr}} = \phi \cdot 1.10 \sqrt{f'_c} \left(\frac{A_{\text{cp}}^2}{p_{\text{cp}}}\right)$$

</div>

* **Combined Shear-Torsion Web Crushing Check ($\phi = 0.75$):**
<div align="center">

$$v_u = \frac{V_u}{b d}, \quad v_t = \frac{T_u p_h}{1.7 A_{oh}^2}$$

$$\sqrt{v_u^2 + v_t^2} \le \phi \left(\frac{V_c}{b d} + 2.1\sqrt{f'_c}\right)$$

</div>

* **Transverse Torsion Reinforcement (per single stirrup leg):**
<div align="center">

$$\frac{A_t}{s} = \frac{T_u}{2 \phi A_o f_{yt} \cot \theta}$$

</div>

* **Maximum Torsion Stirrup Spacing Limit:**
<div align="center">

$$s_{\max,\text{tor}} = \min(p_h / 8, \, 30 \text{ cm})$$

</div>

* **Longitudinal Torsion Reinforcement ($A_l$):**
<div align="center">

$$A_{l,\text{req}} = \left(\frac{A_t}{s}\right) p_h \left(\frac{f_{yt}}{f_y}\right) \cot^2 \theta$$

$$A_{l,\min} = \frac{1.33 \sqrt{f'_c} A_{\text{cp}}}{f_y} - \left(\frac{A_t}{s}\right)_{\text{clamped}} p_h \left(\frac{f_{yt}}{f_y}\right)$$

$$\text{where } \left(\frac{A_t}{s}\right)_{\text{clamped}} = \max\left(\frac{A_t}{s}, \, \frac{1.78 b}{f_{yt}}\right)$$

$$A_l = \max\left(A_{l,\text{req}}, \, \max(0.0, \, A_{l,\min})\right)$$

</div>

</details>

---

## Quick Start & Installation

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
   python concrete_beam_reinforcement_ACI318-14.py
   ```

---

## Workflow & Data Example (`input.json`)

```jsonc
{
  "fc_prime": 240,       // Concrete compressive strength (kg/cm²)
  "fy": 4000,            // Longitudinal steel yield strength (kg/cm²)
  "fy_shear": 4000,      // Stirrup steel yield strength (kg/cm²)
  "b": 30, "h": 60,      // Beam section width and total height (cm)
  "covering": 6,         // Cover to stirrup outer edge (cm)
  "stirrup_dia": 9,      // Stirrup diameter (mm)
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
- **Limit States:** Applies strictly to Ultimate Strength Limit States (ULS). Serviceability checks (immediate/long-term deflections, crack width limits) must be executed separately.

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
- **Engineering Architecture & Validation:** Conceptualized, formulated, and engineered by **ARIYA**. All mathematical formulations, numerical solvers, and verification checks were independently audited against hand calculations.
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