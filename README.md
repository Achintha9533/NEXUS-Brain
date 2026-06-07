

---

# VHT Digital Twin Pipeline - Technical Guide

## Core Component 1: Multimodal Ingestion, PCA Alignment & Survival Modeling

### **Overview**

This section of the pipeline governs the structural harmonization of heterogeneous inputs. It extracts longitudinal multi-sequence MRI radiomics data, synchronizes them with static clinical profiles, and translates them into an orthogonal **Virtual Health Twin (VHT) Signature** via Principal Component Analysis (PCA). This signature drives down-stream ensemble classifiers and semi-parametric Cox Proportional Hazards engines to calculate personalized, outcome-aware baseline patient risk scores.

---

### **Pipeline Flow & Data Processing Stages**

```
 [Excel Multi-Sheet Radiomics] ──> Temporal Parsing (Volumes & Intensities) 
                                                  │
                                                  v
 [Static Patient Clinical Sheet] ─> Row Alignment & Label Encoding (Categoricals)
                                                  │
                                                  v
 [Master Harmonized Matrix] ──────> StandardScaler ──> 99% Variance PCA Reduction
                                                               │
       ┌───────────────────────────────────────────────────────┴──────────────────────────────────────┐
       ▼                                                                                              ▼
 [Gradient Boosting Classifier]                                                            [Cox Proportional Hazards Engine]
 Evaluates structural binary mortality labels (Censored vs. Event)                         Calculates partial hazards to output unique baseline 
 Output: Verification Confusion Matrices                                                   patient risk scores (`risk_score`)

```

---

### **Core Execution Steps & Code Functions**

#### **1. Setup & Longitudinal Radiomic Ingestion**

* **`safe_numeric(s, default)`**: Coerces invalid data strings, dates, or artifacts into NaN floating points to protect data integrity.
* **Multi-Sequence Sheet Extraction**: Parses multiple Excel matrices to extract 10 structural dimensions across four distinct sub-compartments: **Necrotic Core**, **Edema Fluid**, **Enhancing Edge**, and **Resection Cavity**. It tracks volume matrices (`vol`), voxel counts (`vox`), and mean/standard deviation signal metrics for $T_1$, $T_1$-Contrast ($T1c$), $T_2$, and Fluid-Attenuated Inversion Recovery ($FLAIR/T2f$) scans.

#### **2. Multimodal Data Fusion (`patient_objects`)**

* Aggregates longitudinal radiomics timepoints and matches them directly with static molecular phenotypes, staging diagnostics, and demographics from the master clinical sheet.
* Links structural files with underlying spatial folder structures on disk, generating a deeply nested object matrix mapped by uniform patient identifiers (`PID_Clean`).

#### **3. Machine Learning Feature Matrix & PCA Alignment**

* **Feature Pruning**: Drops downstream target event metrics and explicit timeline descriptors to completely eliminate data leakage.
* **Dimensionality Reduction**: Implements an orthogonal transformation via PCA tracking a $99\%$ cumulative variance threshold.
* **Ensemble Classification**: Trains a `GradientBoostingClassifier` ($150$ estimators, max depth $= 4$) on the reduced principal components to robustly verify signal separation between censored individuals and events.

#### **4. Survival Modeling & Risk Stratification (`CoxPHFitter`)**

* Passes the processed numeric features through a semi-parametric Cox Proportional Hazards regression model with $L_2$ Ridge regularization (`penalizer=0.1`) to avoid gradient explosions.
* Evaluates exact coefficients and Hazard Ratios ($HR$) across parameters against long-term diagnostic survival spans.
* **`risk_score` Generation**: Computes continuous personalized partial hazards ($\exp(\beta X)$) per patient. This outcome-aware clinical risk vector is appended directly to the downstream simulation steps to shape personalized kinetic growth trajectories.

---

## Core Component 2: Spatiotemporal Kinetics & Multimodal Feature Engineering

### **Overview**

This module constructs the dynamic longitudinal core of the Virtual Human Twin matrix (`df_kinetics`). It calculates time-dependent tumor boundary velocities, engineers weighted biophysical tissue signals, map clinical molecular profiles, and tracks precise multimodality treatment timelines.

```
 [Longitudinal Patient Objects] ──> Temporal Δt Differencing (Ordered Visits)
                                                │
                                                ├──> Boundary/Voxel Expansion Velocities (Vel_x)
                                                ├──> Heterogeneity & Signal Changes (IntensityChange_x)
                                                └──> Dynamic Treatment Windows & Total Fractions/Doses
                                                │
                                                v
                                  [Master Matrix: `df_kinetics`]

```

---

### **Core Execution Steps & Dynamic Functions**

#### **1. Structural Verification & Dynamic Ordering**

* Sorts chronological imaging visits per patient via an isolated numeric indexer (`get_tp_index`) to properly manage non-uniform diagnostic intervals.
* Enforces strict chronological boundaries ($\Delta t > 0$) to capture true tumor expansion vectors and avoid temporal mismatching.

#### **2. Multi-Compartment Volumetric & Signal Kinematics**

* **`safe_delta(now, prev, dt)`**: Formulates standardized physiological velocity arrays ($\Delta \text{Metric} / \Delta t$) across individual sub-compartments: **Necrotic**, **Edema**, **Enhancing**, and **Resection**.
* **Physiological Mass Changes**: Computes structural boundary speeds (`Vel_x`) and absolute voxel growth rates (`VoxelVel_x`).
* **Microenvironment Evolution**: Incorporates localized scalar metrics evaluating raw structural transformation rates:
* **`IntensityChange_x`**: Combines mean intensities across $T1c$, $T1n$, $T2f$, and $T2w$ using prioritized anatomical weights to detect shifting internal cellular density.
* **`HeterogeneityChange_x`**: Aggregates variance standard deviations to mathematically track emerging necrosis or structural mixed-tissue breakdown over time.



#### **3. Integrated Prior Mapping & Categorical Conditioning**

* **Molecular Profiles**: Injects invariant personal molecular constraints (e.g., *IDH1/2 mutations*, *MGMT methylation status*, *1p/19q co-deletion*, *EGFR amplification*) to establish genetic growth boundaries.
* **Label Alignment**: Implements a vectorized `LabelEncoder` pipeline across categorical markers and clinical stages to create an end-to-end, numeric, ML-ready matrix structure.

#### **4. Continuous Longitudinal Treatment Tracking**

* Maps specific intervention windows by tracking initiation days, completions, and explicit operational durations for Chemotherapy, Radiation, Primary/Secondary Additional plans, Immunotherapy, and Brachytherapy.
* Evaluates treatment-intensity metrics, such as total radiation doses, total delivered fractions, and fractionated dose ratios (`Rad_Dose_Per_Fraction`).
* Generates time-dependent categorical states (`Post_Chemo`, `Post_Rad`, etc.) alongside a unified boolean mask (`TreatmentExposureFlag`) to provide the downstream forecasting engine with explicit clinical intervention timestamps.

---

## Core Component 3: Hybrid Bayesian VHT Forecaster (Mechanistic + CNF Fusion)

### **Overview**

This module couples a deterministic, biophysical tumor growth model with data-driven stochastic estimators. It unifies a **Fisher-KPP partial differential equation (PDE)**, a **Streaming Residual Gaussian Process (GP)**, and a **Continuous Normalizing Flow (CNF)** to map patient trajectory distributions across counterfactual treatment scenarios.

```
                  [Patient Spatial Structural Mask]
                                  │
                                  ▼
               Biophysical Parameter Optimization (Nelder-Mead)
                                  │
                   ┌──────────────┴──────────────┐
                   ▼                             ▼
       [Fisher-KPP PDE Velocity]       [Static Context Conditioning Matrix]
        (Mechanistic Core Model)       (Image Intensities, Genetics, Doses)
                   │                             │
                   ▼                             ▼
       [Streaming Residual GP] ────────> [Conditional Normalizing Flow]
       (Non-parametric Deviations)       (Adjoint ODE Neural Trajectory)
                   │                             │
                   └──────────────┬──────────────┘
                                  ▼
         [Stochastic Scenario-Driven Patient Counterfactuals]

```

---

### **Mathematical & Architectural Foundations**

#### **1. Mechanistic Core: Optimized Fisher-KPP Growth**

Tumor boundary progression is characterized as a reaction-diffusion wavefront. For each patient, the raw volumetric data is mapped to an equivalent spherical tumor radius:


$$r = \left(\frac{3V}{4\pi}\right)^{1/3}$$

The underlying isotropic tumor profile is modeled using the 1D Fisher-KPP radial expansion equation. Over a temporal step $\Delta t$, the radial velocity $dr/dt$ and the corresponding volumetric velocity $dV/dt$ are defined as:


$$\frac{dr}{dt} = \rho r \max\left(0, 1 - \frac{V}{V_{\max}}\right) + 2\sqrt{D\rho}$$

$$\frac{dV}{dt} = 4\pi r^2 \frac{dr}{dt}$$

* **Parameter Optimization**: Patient-specific proliferation rates ($\rho$) and diffusion coefficients ($D$) are derived by passing structural changes through a Nelder-Mead simplex engine minimizing the mean squared error (MSE) between the empirical wave speed and the analytic prediction:

$$\min_{\theta} \frac{1}{N}\sum_{i=1}^{N} \left( \text{Velocity}_{\text{pred}}(\exp(\theta)) - \frac{\Delta r_i}{\Delta t_i} \right)^2, \quad \theta = [\ln\rho, \ln D]$$


* **Carrying Capacity Prior**: $V_{\max}$ is calculated empirically using the 95th percentile of the baseline cohort mask volume scaled by an explicit safety coefficient ($\alpha = 2.0$).

#### **2. Statistical Correction: Streaming Residual GP**

To capture patient-specific physiological anomalies and structural changes not explained by the deterministic PDE, the pipeline fits an online, data-driven residual tracker:


$$y_{\text{residual}} = y_{\text{observed}} - y_{\text{mechanistic}}$$

* **Architecture**: A `MultiOutputRegressor` managing independent Gaussian Process estimators governed by a scaled Radial Basis Function (RBF) kernel:

$$k(x, x') = \sigma_c^2 \exp\left( - \frac{\|x - x'\|^2}{2\ell^2} \right)$$


* **Streaming Engine**: Operates as an online, non-parametric streaming learner. It caches feature arrays dynamically (`X_hist`, `Y_hist`) and executes structural updates sequentially, calculating predictive mean and covariance bounds:

$$\mu_* = K_* N^{-1} y_{\text{residual}}, \quad \Sigma_* = K_{**} - K_* N^{-1} K_*^T, \quad N = K + \sigma_n^2 I$$



#### **3. Generative Flow Neural Odysseys: Conditional Normalizing Flow (CNF)**

Longitudinal multi-tissue dynamics (volumes, signal intensities, and variance trajectories) are structured as continuous-time neural trajectories through a state space $x(t) \in \mathbb{R}^D$, conditioned on an environmental matrix $c \in \mathbb{R}^M$ (composed of genetic priors, baseline risk scores, and temporal intervention logs).

* **Neural ODE Field Engine**: The trajectory is driven by a neural vector field parameterized by an Adjoint Neural ODE framework (`torchdiffeq`):

$$\frac{dx(t)}{dt} = f_{\phi}(x(t), c, t)$$


* **Training Objective**: The network directly learns velocity transformations in log-space ($d_{\log} = \ln(1+x_{t1}) - \ln(1+x_t)$) normalized by empirical statistics ($\mu_{\log}, \sigma_{\log}$). Optimization stabilizes using a standard Euler integration layer ($\Delta t_{\text{step}} = 0.2$) evaluated via MSE:

$$\mathcal{L}(\phi) = \frac{1}{B}\sum_{i=1}^{B} \| f_{\phi}(x_i(t_0), c_i) - d_{\text{log\_scaled}, i} \|^2$$



---

### **Scenario Engine & Counterfactual Trajectory Sampling**

The system projects downstream patient risk states across specific therapeutic horizons ($S_{\text{steps}} = 10$) by dynamically manipulating variables in the conditioning matrix $c$:

| Scenario Type | Conditional Perturbation Target | Operational Parameter Shifts |
| --- | --- | --- |
| **`baseline`** | Standard Natural History | All treatment and timeline fields set to `0.0` |
| **`chemo_on`** | Vectorized Chemotherapy Window | `Post_Chemo` = 1.0; `ChemoDurationDays` = 180.0 |
| **`rad_on`** | Fractionated Radiotherapy Plan | `Post_Rad` = 1.0; Total Dose = 60.0 Gy; Fractions = 30 |
| **`immuno_on`** | Targeted Immunotherapy Cycle | `Post_Immuno` = 1.0; Cycle Length = 14 days; Cycles = 12 |
| **`combined`** | Maximum Multimodal Protocols | Simulates simultaneous chemo, radiation, and immunotherapy |

During counterfactual simulation, individual paths are sampled stochastically using continuous stochastic stepping coupled with scale-adjusted Gaussian perturbations:


$$x_{\log}(t + \Delta t) = x_{\log}(t) + \beta \cdot f_{\phi}(x_{\log}(t), c_{\text{scenario}}) + \gamma \cdot \epsilon, \quad \epsilon \sim \mathcal{N}(0, I)$$

The physical-space trajectory is reconstructed via the inverse mapping $x(t) = \exp(x_{\log}(t)) - 1$, delivering outcome distributions for any simulated clinical path.

---

**Questions**: Contact Kasun Achintha Perera (kasunachintha.perera@studio.unibo.it / pereraachintha84@gmail.com) 🎓