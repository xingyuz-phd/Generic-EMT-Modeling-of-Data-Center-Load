# Generic EMT Modeling of Data Center Load v0 (DC-EMTv0)

**Authors:** Xingyu Zhao, Yingyi Tang, and Junbo Zhao  
**Affiliation:** Dartmouth College  
**PSCAD version:** 5.0.2  
**Last updated:** September 28, 2026  
**Contact:** [xingyu.zhao.th@dartmouth.edu](mailto:xingyu.zhao.th@dartmouth.edu)

## Overview

This project provides a modular electromagnetic transient (EMT) simulation model of data center loads supplied by a centralized uninterruptible power supply (UPS). It is intended for power system studies of control interactions among power electronic modules and dynamic interactions between the data center and the grid.

The PSCAD library, `DC_EMT_lib.pslx`, includes power electronic converters, IT load profiles, control blocks, grid components, and a mechanical load model. Two example cases demonstrate how to assemble a complete data center model and compare converter models at different fidelity levels.

## Workspace

| Workspace item | Description |
| --- | --- |
| `DC_EMT_ws` | Workspace containing the component library and example projects |
| `DC_EMT_lib.pslx` | Modular data center EMT component library |
| `DC_EMT_demo` | Complete data center example using average converter models |
| `DC_EMT_compare` | Comparison of model responses at different fidelity levels |

## PSCAD Version and Required Settings

The supplied project files use **PSCAD 5.0.2**, as shown by both the Product Version and File Version in the project settings.

> [!IMPORTANT]
> **Do not enable “Use ideal branches for resistances under”.**
> In **Project Settings → Runtime → Network Solution Accuracy**, leave this checkbox **unchecked** before running the model.
> Apply this setting to both `DC_EMT_demo` and `DC_EMT_compare`, and to any new project using these library modules.

## Component Library

![DC_EMTv0 component library](dc_emt_component_library.png)

*Figure 1. Power electronic modules and supporting components in `DC_EMT_lib.pslx`.*

### Power Electronic Modules

| Module | Average model (AVM) | Switching model (SWM) | Reduced-order model (ROM) |
| --- | :---: | :---: | :---: |
| Rectifier | Yes | Yes | Yes |
| Inverter | Yes | Yes | Yes |
| Battery DC–DC converter | Yes | Yes | Yes |
| Power factor correction (PFC) converter | Yes | Yes | Yes |
| Buck converter | Yes | Yes | Yes |
| Variable frequency drive (VFD) | Yes | Yes | — |

- **AVM:** Represents average converter behavior while retaining circuit and control dynamics.
- **SWM:** Represents switching devices and PWM operation, including switching ripple.
- **ROM:** Simplifies fast dynamics while retaining the outer controls and energy-storage dynamics represented by each module.

Parameters retained in the ROM use the same values as in the corresponding SWM.

### Supporting Modules

- **IT load profiles:** Ramp, sinusoidal, and square-wave demand variations.
- **Control blocks:** PI controller, UPS supervisory control, and voltage ride-through (VRT) control.
- **Grid components:** Grid equivalent and voltage-dip profile.
- **Mechanical load:** Shaft-load model for motor-driven cooling loads.

## Control Methods

The following table summarizes the control objectives and structures. AVM and SWM retain the explicit current loops where used; ROM replaces these fast dynamics with ideal current tracking or an algebraic current response while retaining the represented outer controls.

| Module | Control method |
| --- | --- |
| Rectifier | **Grid-following (GFL) control.** A PLL tracks the grid angle. An outer DC-voltage PI generates the d-axis current reference, while the q-axis reference is zero. Inner dq current loops regulate the AC current. |
| UPS inverter | **Grid-forming (GFM) control at the UPS output.** Cascaded voltage and current loops establish the protected AC voltage using an internally generated angle. A slow phase-alignment loop aligns this angle with the upstream PLL to reduce phase mismatch during transfers involving the bypass. |
| Battery DC–DC converter | **Mode-dependent control.** Optional DC-voltage droop provides support during online operation. In battery mode, a voltage PI takes over regulation of the UPS DC link. A recovery ramp smooths the return to online operation. |
| PFC converter | **DC-voltage regulation and input-current shaping.** The voltage PI produces a conductance command. Multiplying it by the rectified input voltage generates the current reference, targeting near-unity power factor under sinusoidal supply conditions. |
| Buck converter | **Cascaded voltage and current control.** The voltage PI generates an inductor-current reference, and the current PI produces the duty command. A time-varying load resistance represents workload changes downstream of the regulated DC supply. |
| VFD | **Open-loop V/F control.** The drive sets the inverter frequency from the command and adjusts voltage magnitude according to a V/F profile. Rotor speed is not fed back for speed regulation, so the actual speed depends on motor slip and mechanical loading. |
| UPS supervisory and VRT controls | **Operating-mode selection.** Grid-voltage conditions and the ride-through logic determine mode commands; the UPS supervisor coordinates the rectifier, inverter, and bypass breakers. |

The VFD's V/F profile aims to maintain approximately constant motor flux in its constant-ratio operating region. For background, see Texas Instruments' [Scalar (V/f) Control of 3-Phase Induction Motors](https://www.ti.com/lit/an/sprabq8/sprabq8.pdf).

## Bandwidth-Based PI Tuning

The PI-regulated loops use a second-order target to relate response speed and damping to controller gains. Instead of selecting gains independently, specify a tuning frequency $f_{\mathrm{bw}}$ in hertz and a damping ratio $\zeta$.

For a first-order plant approximation and a parallel PI controller,

$$
G(s)=\frac{K}{as+b},\qquad C_{\mathrm{PI}}(s)=K_p+\frac{K_i}{s},
$$

where $a>0$ and $K>0$, negative unity feedback gives the characteristic polynomial

$$
a s^2+(b+K K_p)s+K K_i.
$$

Matching this to $a(s^2+2\zeta\omega_n s+\omega_n^2)$ gives

$$
\omega_n=2\pi f_{\mathrm{bw}},\qquad
K_p=\frac{2\zeta\omega_n a-b}{K},\qquad
K_i=\frac{a\omega_n^2}{K}.
$$

A higher tuning frequency gives faster target dynamics; the damping ratio sets the damping of the target poles. The coefficients $a$, $b$, and $K$ must represent the particular loop, including its per-unit scaling and operating point.

For example, a capacitor voltage loop with ideal inner current tracking has $a=C_{\mathrm{pu}}/\omega_b$, $b=0$, and $K=1$, when the controller commands capacitor-side current and load current is treated as a disturbance. With physical time in seconds and $\omega_b=2\pi f_{\mathrm{base}}$,

$$
K_p=\frac{2\zeta\omega_n C_{\mathrm{pu}}}{\omega_b},\qquad
K_i=\frac{\omega_n^2 C_{\mathrm{pu}}}{\omega_b}.
$$

Tune inner current loops first, then choose slower outer voltage loops so that the fast-current-loop approximation is reasonable. Verify the resulting response with the connected converter chain, including filters, switching delays, and limits.

**Interpretation of bandwidth:** The parameter labeled “bandwidth” sets the target natural frequency through $\omega_n=2\pi f_{\mathrm{bw}}$. It is not generally equal to the measured closed-loop −3 dB bandwidth: the PI zero and additional dynamics also affect the response. ROM uses the same tuning values as SWM for every retained controller; tuning inputs for removed loops are omitted.

## Example Cases

### Case 1: Complete Data Center Model (`DC_EMT_demo`)

This case demonstrates how to use the modules in `DC_EMT_lib.pslx` to build a complete data center model for power system dynamic studies. It uses average converter models and includes disturbances on both the grid and workload sides.

![Example data center load model](dc_emt_data_center_example.png)

*Figure 2. Example data center model with UPS-supplied IT loads, VFD-driven cooling loads, and static lighting loads.*

The simulation sequence is as follows:

1. **Initialization (t = 0–5 s):** The IT load is initialized to 0.6 pu. The cooling load, represented by an induction motor, accelerates toward steady operation under the commanded V/F profile.

2. **Grid-side disturbance (t = 5 s):** A voltage dip is applied at 5 s, and the voltage begins to recover at 6.5 s. This event illustrates the responses of the data center components, particularly the UPS, to the voltage dip and recovery according to the implemented voltage ride-through logic.

3. **Workload-side disturbance (t = 12 s onward):** A periodic square-wave variation is applied to the IT workload to represent power demand fluctuations associated with AI training tasks.

### Case 2: Model Fidelity Comparison (`DC_EMT_compare`)

This case compares the responses of the power electronic modules in `DC_EMT_lib.pslx` at different modeling fidelity levels.

#### IT Loads

The average, switching, and reduced-order models are compared under the following simulation sequence:

1. **Initialization (t = 0–2 s):** The IT load is initialized to 0.6 pu.

2. **Grid-side disturbance (t = 2 s):** A voltage dip is applied at 2 s, and the voltage begins to recover at 2.5 s.

3. **Workload-side disturbance (t = 6 s onward):** A periodic square-wave variation is applied to the IT workload to represent power demand fluctuations associated with AI training tasks.

These disturbances allow the three model variants to be compared under both grid-side and workload-side changes.

#### Cooling Loads

The cooling-load model is initialized during the first 5 s. At 5 s, the voltage-dip profile used for the IT-load comparison is applied to the cooling-load supply. This test compares the responses of the average and switching models to a grid-side disturbance.

Together, these comparisons illustrate how modeling fidelity affects simulated dynamic responses and support model selection for different study objectives.

## Citation

If you use this model in your research, please cite the software repository:

> X. Zhao, Y. Tang, and J. Zhao, *Generic EMT Modeling of Data Center Load v0 (DC_EMTv0)*, version v0, Dartmouth College, 2026. [Software]. Available: [GitHub repository](https://github.com/xingyuz-phd/Generic-EMT-Modeling-of-Data-Center-Load).

### BibTeX

```bibtex
@misc{zhao2026dcemt,
  author       = {Zhao, Xingyu and Tang, Yingyi and Zhao, Junbo},
  title        = {Generic {EMT} Modeling of Data Center Load v0 ({DC\_EMTv0})},
  year         = {2026},
  howpublished = {GitHub repository},
  url          = {https://github.com/xingyuz-phd/Generic-EMT-Modeling-of-Data-Center-Load},
  note         = {Version v0, Dartmouth College}
}
```

## Contact

For questions about the model, please contact Xingyu Zhao at [xingyu.zhao.th@dartmouth.edu](mailto:xingyu.zhao.th@dartmouth.edu).
