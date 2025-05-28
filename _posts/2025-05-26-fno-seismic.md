---
layout: distill
title: Are Fourier Neural Operators Really Faster?
description: We challenge the performance benefits of Fourier Neural Operators, particularly for linear wave propagation problems
tags: distill formatting
giscus_comments: true
date: 2025-05-26
featured: true

authors:
  - name: Dimitri Voytan
    url: "https://en.wikipedia.org/wiki/Albert_Einstein"
    affiliations:
      name: Avathon, Pleasonton California
  # - name: Boris Podolsky
  #   url: "https://en.wikipedia.org/wiki/Boris_Podolsky"
  #   affiliations:
  #     name: IAS, Princeton

# bibliography: 2018-12-22-distill.bib

# Optionally, you can add a table of contents to your post.
# NOTES:
#   - make sure that TOC names match the actual section names
#     for hyperlinks within the post to work correctly.
#   - we may want to automate TOC generation in the future using
#     jekyll-toc plugin (https://github.com/toshimaru/jekyll-toc).
toc:
  - name: Introduction
    # if a section has subsections, you can add them as follows:
    # subsections:
    #   - name: Example Child Subsection 1
    #   - name: Example Child Subsection 2

# Below is an example of injecting additional post-specific styles.
# If you use this post as a template, delete this _styles block.
_styles: >
  .fake-img {
    background: #bbb;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  }
  .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }
---


## Introduction

Fourier Neural Operators (FNOs) are often advertised as fast, mesh-invariant alternatives to traditional PDE solvers, with some studies claiming massive speedups over finite-difference (FD) methods. But do those claims hold up when we put FNOs head-to-head with optimized, GPU-accelerated FD codes?

In this post, we benchmark a state-of-the-art Tucker-tensorized FNO against hand-tuned FD solvers for four seismic wave physics formulations: isotropic acoustic, TTI-acoustic, isotropic elastic, and VTI-elastic. What we find is surprising: FNOs are usually slower—often *much* slower—unless you're doing a very specific type of reduced-order modeling.

<image></image>

## Background: Solving Wave Equations

Seismic wave simulations solve PDEs that model how waves travel through the Earth. One classic example is the 3D acoustic wave equation:

$\frac{\partial^2 u}{\partial t^2} = c^2 \nabla^2 u$

FD methods discretize space and time into a grid, then update the wavefield with local stencils. They're fast, parallelizable, and well-understood. But they get exponentially more expensive as you increase frequency, lower velocity, or demand high accuracy—you need finer grids and smaller time steps due to the Courant-Friedrichs-Lewy (CFL) condition and numerical dispersion.

On the other hand, FNOs aim to learn a mapping from input fields (like velocity) to output fields (like pressure or displacement) using deep neural networks. Crucially, FNOs work in the Fourier domain and apply learned convolutions via FFTs.

<image></image>

## Experiments

We ran both FNOs and FD solvers on the same hardware (NVIDIA A100s) and carefully controlled for fair benchmarking:

* Same simulation domain size
* Same time duration (1/2 second)
* Same frequency content (30 Hz max)
* FD grid spacing = 10m (limited by points per wavelength)
* FNO spacing = 10m and 25m (Nyquist frequency)
* Varied model complexity: from simple acoustic to full anisotropic elasticity

We didn't train any models. Instead, we focused only on *inference time* — the cost of running the forward pass once the model is trained.

## Key Results

### 1. On identical grids (10m spacing), FNOs are *always* slower.

| Physics     | Small FNO (4 Layers, 12 Channels) | Medium FNO (8 Layers, 24 Channels) | Large FNO (12 Layers, 36 Channels) |
| ----------- | ------------------- | -------------------- | -------------------- |
| Acoustic    | 1.4x *slower*       | 27x *slower*         | 80x *slower*         |
| VTI Elastic | 1.4x *slower*       | 3x *slower*          | 1.3x *slower*        |

### 2. If we allow the FNO to use a coarser 25m grid, it can catch up.

| Physics     | Small FNO      | Medium FNO     | Large FNO      |
| ----------- | -------------- | -------------- | -------------- |
| Acoustic    | 1.25x *faster* | 2.7x *slower*  | 6x *slower*    |
| VTI Elastic | 9.65x *faster* | 2.89x *faster* | 1.30x *faster* |

Only the smallest networks (4 layers, 12 channels) achieve consistent speedups. Bigger models, even on coarse grids, are still slower.

<image></image>

### 3. The real win? Predicting *only the surface* wavefield.

If you restrict the FNO to predicting the full time history at a single depth (like the surface), it avoids recurrent inference and runs extremely fast.

| Model      | Speedup over FD |
| ---------- | --------------- |
| FNO-Small  | 4200x           |
| FNO-Medium | 1260x           |
| FNO-Large  | 568x            |

<image></image>

## Why Are FNOs Slower?

The math is clear:

* FD scales as $O(p^d \beta^d N)$
* FNO scales as $O(L N \log N)$

Where:

* $p$ = stencil width
* $\beta$ = oversampling factor
* $N$ = number of points
* $L$ = number of FNO layers

In 3D, $p^d \beta^d$ grows fast, but not fast enough to offset the heavy lifting of FFTs unless the FNO network is *very* shallow.

Also, FNOs carry a lot of constant-factor overhead: memory pressure, activation storage, and FFT setup costs.

## When Are FNOs Useful?

They *are* valuable in some cases:

* **Receiver-only predictions** (e.g., marine acquisition)
* **Low-resolution surrogate models** where full fidelity isn't needed

But for full-volume, high-res, time-domain simulations? They're not competitive yet.

## Takeaways

* FNOs are not faster than FD solvers for full-volume simulations
* Even on coarser grids, FNOs must be small to offer speedups
* Major gains come only when predicting surface fields (i.e., output dimensionality is reduced)

If you're designing an FNO for wave simulations, benchmark it properly against a tuned FD code. And if you're aiming for performance, remember that "learned" doesn't always mean "faster."

<image></image>

## Read More

* [Devito finite difference framework](https://www.devitoproject.org/)
* [Fourier Neural Operator original paper](https://arxiv.org/abs/2010.08895)
* [Tucker-Tensorized FNO](https://arxiv.org/abs/2310.00120)

---