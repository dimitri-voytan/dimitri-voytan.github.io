---
layout: distill
title: Are Fourier Neural Operators Really Faster?
description: I challenge the performance benefits of Fourier Neural Operators, particularly for linear wave propagation problems
tags: NeuralOperators Ai4Science
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
  - name: Background, Solving Wave Equations
  - name: Experiments
  - name: Key Results
  - name: Why Are FNOs Slower?
  - name: When Are FNOs Useful?
  - name: Takeaways
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

Fourier Neural Operators (FNOs) are often advertised as fast, mesh-invariant alternatives to traditional PDE solvers, with some studies claiming orders of magnitude speedups over classical methods, like finite-difference (FD). But do those claims hold up when we put FNOs head-to-head with optimized, GPU-accelerated FD codes? I've spent more time than I'm willing to admit trying to pin this down.

In this post, I benchmark a state-of-the-art Tucker-tensorized FNO against hand-tuned FD solvers for four seismic wave physics formulations: isotropic acoustic, TTI-acoustic, isotropic elastic, and VTI-elastic. What I find is not altogether surprising: FNOs are usually slower—often *much* slower—unless you do some type of reduced-order modeling, like taking extremely large time steps or limiting the solution to just a surface.

In this post I'm focusing primarily on seismic wave propogation problems, but the results *probably* extend to other linear PDEs of similar complexity. It would be straightforward to extend the code for that purpose.

## Background, Solving Wave Equations

Seismic wave simulations solve PDEs that model how waves travel through the Earth. One classic example is the 3D acoustic wave equation:

$\frac{\partial^2 u}{\partial t^2} = c^2 \nabla^2 u$

Finite difference methods discretize space and time into a grid, then update the wavefield with local stencils. They're fast, parallelizable, and well-understood. But they get exponentially more expensive as you increase frequency, lower velocity, or demand high accuracy—you need finer grids and smaller time steps due to the Courant-Friedrichs-Lewy (CFL) condition and numerical dispersion.

On the other hand, FNOs can be trained to learn a mapping from input fields (like velocity) to output fields (like pressure or displacement) using deep neural networks. This is the same "information" provided by the finite difference simulation. Crucially, FNOs work in the Fourier domain and apply learned convolutions via FFTs.


## Experiments

I ran both FNOs and FD solvers on the same hardware (NVIDIA A100s) and carefully controlled for fair benchmarking:

* Same simulation domain size
* Same time duration (1/2 second)
* Same frequency content (30 Hz max)
* FD grid spacing = 10m (limited by points per wavelength)
* FNO spacing = 10m and 25m and 60Hz sample rate (Nyquist frequency in space and time)
* Varied model complexity: from simple acoustic to full anisotropic elasticity
* Ensure only the forward pass cost is recorded. No memory access latency.

I didn't train any models. Instead, I focused only on *inference time* — the cost of running the forward pass once the model is trained. The run time doesn't depend on the specific weights, so you can save a lot of time by avoiding training. 

To ensure a fast finite difference solver, I used [Devito](https://www.devitoproject.org/). Devito generates optimized explicit finite difference codes from symbolically described PDE. It offers several performance benefits, including GPU acceleration, "out-of-the box". I ran all simulations in 3 spatial dimensions, which is representative of the PDE simulations carried out today.

## Key Results

### 1. On identical grids (10m spacing), FNOs are *always* slower.

| Physics     | Small FNO (4 Layers, 12 Channels) | Medium FNO (8 Layers, 24 Channels) | Large FNO (12 Layers, 36 Channels) |
| ----------- | ------------------- | -------------------- | -------------------- |
| Acoustic    | 1.4x *slower*       | 27x *slower*         | 80x *slower*         |
| VTI Elastic | 1.4x *slower*       | 3x *slower*          | 1.3x *slower*        |

The figure below gives a more comprehensive view of the performance characteristics. I fit a linear regression to the runtimes and report the slope ratio (speedup, assuming the y-intercept is 0). FNO is in blue and the other types of wave propagation, simulated with a finite difference solver, have their own color respectively. 

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/posts/fno_seismic/fno_10_10_10.png" class="img-fluid rounded z-depth-1 zoomable=true " %}
    </div>
</div>
<div class="caption">
  Runtime scaling when FNO and Finite difference are run on the same spatial grid (10 x 10 x 10m). I assume a temporal Nyquist frequency for FNO and the CFL limit for finite difference. The reported slope ratios indicate the speedup, relative to FNO. 
</div>

### 2. If we allow the FNO to use a coarser 25m grid, it can catch up.

| Physics     | Small FNO      | Medium FNO     | Large FNO      |
| ----------- | -------------- | -------------- | -------------- |
| Acoustic    | 1.25x *faster* | 2.7x *slower*  | 6x *slower*    |
| VTI Elastic | 9.65x *faster* | 2.89x *faster* | 1.30x *faster* |

Only the smallest networks (4 layers, 12 channels) achieve consistent speedups. Bigger models, even on coarse grids, are still slower. In this regime, I believe that the risk of poor inference quality negates their benefits.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/posts/fno_seismic/fno_25_25_25.png" class="img-fluid rounded z-depth-1 zoomable=true " %}
    </div>
</div>
<div class="caption">
  Here, I've coarsened the FNO grid to 25 x 25 x 25 m and left the finite difference on the same grid.
</div>

### 3.  Predicting *only the surface* wavefield.

If you restrict the FNO to predicting the full time history at a single depth (like the surface), it avoids recurrent inference and runs extremely fast. This was an approach proposed by [Lehman (2024).](https://www.sciencedirect.com/science/article/abs/pii/S0045782523008411) 

| Model      | Speedup over FD (VTI elastic) |
| ---------- | --------------- |
| FNO-Small  | 4200x           |
| FNO-Medium | 1260x           |
| FNO-Large  | 568x            |


<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/posts/fno_seismic/surface_wavefield.png" class="img-fluid rounded z-depth-1 zoomable=true " %}
    </div>
</div>
<div class="caption">
  Here, I've predicting only the surface wavefield with FNO.
</div>

## Why Are FNOs Slower?

If you consider just a single timestep:

* FD scales as $O(p^d \beta^d N)$
* FNO scales as $O(L N \log N)$

Where:

* $p$ = spatial derivative stencil width
* $\beta$ = oversampling factor (relative to Nyquist)
* $N$ = number of points in the domain
* $L$ = number of FNO layers

In 3D, $p^d \beta^d$ grows fast, but not fast enough to offset the heavy lifting of FFTs unless the FNO network is *very* shallow.

## When Are FNOs Useful?

They *are* valuable in some cases:

* **Receiver-only predictions** (e.g., marine acquisition)
* **Low-resolution surrogate models** where full fidelity isn't needed

But for full-volume, high-res, time-domain simulations? They're not competitive yet.

Some caveats here. I optimized both finite difference and FNO codes reasonably well (compiling static graphs, turning off batch normalization, setting to inference mode, etc.), but optimization can he a never ending pit of work. I made a reasonably good effort to improve performance, but not exhaustive. Also, I didn't consider half-precision runs e.g. `fp16` or `bf16`. It was not straightforward to get working in the FNO FFTs and support was only recently added to Devito. But the results shouldn't change fundamentally if both could be run at half precision. 

## Takeaways

* FNOs are not faster than FD solvers for full-volume simulations
* Even on coarser grids, FNOs must be small to offer speedups
* Major gains come only when predicting surface fields (i.e., output dimensionality is reduced)

If you're designing an FNO for any linear PDE, benchmark it properly against a tuned FD code. And if you're aiming for performance, remember that "learned" doesn't always mean "faster."

<image></image>

## Read More

* [Code (Coming Soon!)](https://github.com)
* [Paper (Still in Review, Preprint Soon)](https://arxiv.org)
* [Devito finite difference framework](https://www.devitoproject.org/)
* [Fourier Neural Operator original paper](https://arxiv.org/abs/2010.08895)
* [Tucker-Tensorized FNO](https://arxiv.org/abs/2310.00120)

---