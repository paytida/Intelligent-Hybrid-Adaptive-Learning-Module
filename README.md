# Intelligent Hybrid Adaptive Learning Module using Convex Combination & Subband Filtering

An implementation of a block-based MATLAB Simulink architecture for real-time identification of mixed linear, nonlinear, and spectrally colored dynamical systems.

This project integrates standard Normalized Least Mean Squares (NLMS) linear filtering, Functional Link Artificial Neural Networks (TFLANN) for nonlinear expansion, Subband Adaptive Filtering (SAF) for spectrally colored environments, and an adaptive convex combination mechanism driven by gradient descent.

---

## Key Features

* **Block-Based Simulink Architecture:** Built using standard Simulink library blocks without embedded programmatic functions or external scripts for core adaptive routing, ensuring deterministic execution and hardware compatibility.


* **Trigonometric FLANN (TFLANN) Expansion:** Expands scalar inputs into a higher-dimensional functional basis (polynomial and trigonometric) to approximate arbitrary system non-linearities without hidden-layer backpropagation overhead.


* **Adaptive Convex Combination:** Dynamically blends linear and nonlinear filter branches using an online gradient-descent update on the mixing parameter $\lambda(k) \in [0, 1]$.


* **Subband Adaptive Filtering (SAF):** Utilizes multi-band analysis/synthesis filter banks to decompose input signals into localized subbands, mitigating slow convergence caused by high eigenvalue spreads in correlated noise.


* **Embedded Hardware Validation:** Deployed and validated on real-time hardware (DSP/FPGA platforms via C/RTL code generation) with resistance to fixed-point quantization noise and timing jitter.



---

## Mathematical Architecture

### 1. Model Formulation & Convex Output

For an input $x(k)$ and desired output $d(k) = \mathcal{H}\{x(k)\} + \eta(k)$:


$$y(k) = \lambda(k) y_L(k) + (1 - \lambda(k)) y_{NL}(k)$$

where:

* $y_L(k) = \mathbf{w}_L^T(k) \mathbf{x}(k)$ is the linear filter output.


* $y_{NL}(k) = \mathbf{w}_{NL}^T(k) \boldsymbol{\phi}(k)$ is the nonlinear TFLANN output.


* $\lambda(k) \in [0, 1]$ is the adaptive mixing weight.



### 2. Trigonometric FLANN Basis

The scalar input $x(k)$ is mapped into a nonlinearly expanded feature vector $\boldsymbol{\phi}(k)$:


$$\boldsymbol{\phi}(k) = \left[ x(k), x^2(k), x^3(k), \sin(\pi x(k)), \cos(\pi x(k)), \sin(2\pi x(k)), \cos(2\pi x(k)), \dots \right]^T$$

### 3. Adaptive Update Equations

The overall instantaneous system error $e(k) = d(k) - y(k)$ drives all concurrent weight updates:

| Parameter | Update Equation | Description / Step Size |
| --- | --- | --- |
| **Linear Weights ($\mathbf{w}_L$)** | $\mathbf{w}_L(k+1) = \mathbf{w}_L(k) + \frac{\mu_L}{\Vert{}\mathbf{x}(k)\Vert{}^2 + \epsilon} e(k) \mathbf{x}(k)$ | Normalized LMS (NLMS) update rule. |
| **Nonlinear Weights ($\mathbf{w}_{NL}$)** | $\mathbf{w}_{NL}(k+1) = \mathbf{w}_{NL}(k) + \mu_{NL} e(k) \boldsymbol{\phi}(k)$ | Functional Link LMS update rule. |
| **Mixing Parameter ($\lambda$)** | $\lambda(k+1) = \text{sat}\Big( \lambda(k) + \mu_\lambda e(k) \big(y_L(k) - y_{NL}(k)\big) \Big)$ | Gradient descent on $\frac{1}{2} e^2(k)$, saturated to $[0, 1]$. |

---

## Implemented Hybrid Architectures

Three distinct parallel architectures were constructed in Simulink to evaluate trade-offs between nonlinear modeling capacity and spectral whitening performance:

1. **Architecture 1 (NLMS + TFLANN):** Baseline full-band linear NLMS combined with a full-band Trigonometric FLANN module. Ideal for spectrally flat (white) inputs with strong harmonic distortion.
2. **Architecture 2 (NLMS + Subband NLMS):** Combines a full-band NLMS filter with a 16-band Subband Adaptive Filter (SAF) bank.
3. **Architecture 3 (Subband NLMS + Subband TFLANN):** Combines a subband linear architecture with subband nonlinearly expanded TFLANN filters across decomposed channels.



---

## System Performance & Stress Tests

### Software Simulation Analysis

```
       [Input Noise Stress: White / Pink / Brown]
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
    Linear Branch (NLMS/SAF)    Nonlinear Branch (TFLANN)
             │                           │
             └─────────────┬─────────────┘
                           ▼
              Adaptive Convex Combiner
                     λ(k) Update
                           │
                           ▼
                    Combined Output

```

* **Gaussian White Noise ($\chi(\mathbf{R}) \approx 1$):** Architecture 1 (NLMS + TFLANN) yields the fastest initial error convergence due to exact trigonometric basis alignment without filter bank group delay overhead.
* **Pink Noise ($1/f$ Spectrum):** Moderate eigenvalue spread degrades standard NLMS convergence. Subband architectures (Arch 2) take over as localized subband whitening restores fast adaptation rates.
* **Brown Noise ($1/f^2$ Spectrum):** Severe low-frequency energy concentration creates extreme eigenvalue spread. The adaptive parameter $\lambda(k)$ shifts reliance entirely toward the Subband branch, preventing filter divergence and maintaining low steady-state MSE.



---

## On-Device Hardware Validation

The complete block-based model was targeted for real-time execution on DSP / FPGA hardware via Simulink code generation.
* **Fixed-Point Quantization Resilience:** Hardware execution using fixed-point word lengths introduces rounding noise in weight updates; despite this, the adaptive convex combination mechanism retains low steady-state MSE.
* **Parameter Drift & Stability:** Real-world hardware stress tests show $\lambda(k)$ dynamically tracking non-stationary environmental variations and I/O noise while remaining bounded within $[0, 1]$.

---

## Requirements & Setup

1. **Software:**
* MATLAB R2023b or newer
* Simulink
* Signal Processing Toolbox
* DSP System Toolbox (for Subband Analysis/Synthesis Filter Banks)

2. **Hardware Deployment (Optional):**
* Embedded Coder / HDL Coder
* Supported DSP (e.g., TI C2000/C6000) or FPGA target board
