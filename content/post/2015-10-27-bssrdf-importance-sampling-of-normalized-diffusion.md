---
categories: []
coverImage: images/rlShaders/rlSssNd_buddha.jpg
date: 2015-10-27T00:00:00Z
image: images/header/posts/2015-10-27-bssrdf-importance-sampling-of-normalized-diffusion.jpg
published: true
tags:
- Arnold
- BSSRDF
- Importance Sampling
title: BSSRDF Importance Sampling of Normalized Diffusion
---

Highly scattering materials like milk or skin can't be precisely modeled by BRDF only, because light would scatter multiple times before exiting the material at different locations. Bidirectional surface scattering distribution function (BSSRDF) is used to describe radiance transfer across the surface boundary with diffusion profile for multiple scattering events.

To solve subsurface light transport efficiently, there are several approaches to model and evaluate the diffusion profile. Recently, Burley [[5](#ref.5)] has proposed an artist-friendly diffusion profile with _only two_ exponential functions and the most attracting thing is that it's not only intuitive for artists to use but also faster to evaluate. It's a great exercise to try it out for fun in Arnold.

# BSSRDF & Diffusion Profile

<img src="/images/rlShaders/bssrdf_scatter.png" style="box-shadow: none;">

BSSRDF is a function relates outgoing radiance to the incident flux at different points:

$$S(x_i,\vec{\omega_i};x_o,\vec{\omega_o})=\frac{dL_0(x_o,\vec{\omega_o})}{d\phi_i(x_i,\vec{\omega_i})}$$

For efficiency, $S$ can be decomposed into single-scattering and multiple-scattering terms. The multiple scatter term $S_d$ is too complex to solve directly, the common strategy is to use a `diffusion profile` to approximate the distribution of outgoing radiance. With the diffusion approximation to the _radiative transport equation_ (RTE), $S_d$ is represented by a product of two Fresnel refraction terms and a diffusion profile in BSSRDF:

$$S_d(x_i,\vec{\omega_i};x_o,\vec{\omega_o})=\frac{1}{\pi}F_t(\eta,\vec{\omega_i})R_d(\vert x_i-x_o\vert)F_t(\eta,\vec{\omega_o})$$

and the subsurface light transport is formulated as:

<div>$$L_o(x_o,\vec{\omega_o})=\int_A{\int_\Omega{S(x_i,\vec{\omega_i};x_o,\vec{\omega_o}) L_i(x_i,\vec{\omega_i})(n\cdot\vec{\omega_i}) d\omega_i}dA(x_i)}$$</div>

> There are many technical terms among the academic papers, such as fluence, mean free path, etc. I found the best introductory reading materials to understand the relevant details of sub-surface scattering approximation are:
>
> * [A Practical Model for Subsurface Light Transport](https://graphics.stanford.edu/papers/bssrdf/). Jensen et al. 2001.
> * [Towards Realistic Image Synthesis of Scattering Materials](http://www.cs.jhu.edu/~misha/Fall11/Donner.Thesis.pdf). Donner. 2006.
> * [Classical and Improved Diffusion Theory for Subsurface Scattering](http://graphics.pixar.com/library/PhotonBeamDiffusion/index.html). Ralf Habel et al. 2013.
>
> About the diffusion profiles, there are several profiles such as diploe, multipole, quantized diffusion and directional dipole. The presentation [slides](http://www.ci.i.u-tokyo.ac.jp/~hachisuka/dirpole_slides.pdf) of directional profile [J. R. Frisvad et al. 2014] has great illustrations for the integration scheme of each diffusion profile.

# BSSRDF Importance Sampling Framework in Arnold

To solve the subsurface light transport, two primary approaches are used in graphics community. One is a multi-pass technique, which uses point cloud over the surface to store the diffuse irradiance at each point in pre-processing stage, then utilize that irradiance cache to solve the light transport equation in second phase.

<img src="/images/rlShaders/bssrdf_sampling.png" style="box-shadow: none;">

The other is to perform importance sampling of the diffusion profile directly. Without the needs of point cloud generation, this approach fits well in any path-tracer with probe tracing API. The state-of-the-art sampling method proposed by King et al. [[4](#ref.4)] can be summarized to following steps:

1. sample a point on disk with PDF in proportion to the diffusion profile $R(r\)$
2. randomly select probe orientation with local basis U, V, N at $x_o$
3. compute probe origin and direction from the selected orientation
4. trace probe rays to shade all hit points within a range of max radius
5. combine sample results with MIS (Multiple Importance Sampling):

    <div>$$w_i=\frac{R_d(\|x_o-x_i\|)}{\sum_j{c_j\ pdf_{disk}(r_j) |\vec{n_o}\cdot\vec{n_i}|}}$$</div>

    where $c_j$ is the sample weight of probe axis selection (N:0.5, U:0.25, V:0.25), and $r_j$ is the projected radius on UV, NV, NU plane.

Arnold SDK only provides APIs for Gaussian and cubic profile currently, for other profiles, we have to implement the sampling routine from scratch. The implementation details would be discussed in later section. Now, let's see how to sample the radius $r$ from the normalized diffusion profile $R(r\)$.

# Importance Sampling of Normalized Diffusion

_Normalized diffusion_ profile [[5](#ref.5)][[6](#ref.6)] is formulated by two exponential functions:

<div>$$R(r)=\frac{e^{−r/d}+e^{−r/3d}}{8\pi dr}$$</div>

and the beauty of this model is that with <span class="green">__any positive value of $d$ the integration over the disk is always one:__</span>

<div>$$\int_0^\infty{\int_0^{2\pi}{R(r)r\ d\phi}\ dr}=\int_0^\infty{R(r)2\pi r\ dr}=1$$</div>

(The parameter $d$ can be interpreted as the scattering distance for artistic control. For its relationship to other physical quantities like surface albedo and mean free path, please refer to [[6](#ref.6)])

Although $R(r\)$ is surprising simple, but its CDF is not analytically invertible. Therefore, I try to randomly select one exponential and generate sample $r$ from it, then use MIS to combine the sample weights. Besides, in order to make probe tracing faster and to reduce the variance (noise), it's better to restrict the integration domain within a user defined max radius $R_{max}$.

## Lobe selection

There are two exponential functions and three distance values $d$ for R, G, B. In other words, there are <span class="orange">__six lobes__</span> to sample. Before sampling the radius $r$ on disk, we need to select a lobe in two steps:

1. choose the value of $d$ from R, G, B <span class="orange">__uniformly__</span>
    * since every positive $d$ would always get one for the integral of $R(r )$
2. given $d$, use the ratio of integration within $R_{max}$ to select the exponential function:

<div>$$w_1=1-e^{-R_{max}/d}, w_2=3(1-e^{-R_{max}/{3d}})$$</div>

## Sample $r$ from a lobe

Suppose the PDF of the first exponential is $p\_1(r\)\propto \frac{e^{−r/d}}{8\pi dr}$, and it's normalized in range $[0, R_{max}]$

<div>$$\int{c\frac{e^{-r/d}}{8\pi dr}2\pi r\ dr}=\int{\frac{c}{4d}e^{-r/d}\ dr}=\frac{-c}{4}(e^{\frac{-R_{max}}{d}-1})=1 \\
c=\frac{4}{1-e^{-R_{max}/d}},\ p_1(r)=\frac{e^{-r/d}}{2\pi dr}\frac{1}{1-e^{-R_{max}/d}}$$</div>

then the CDF of $p_1(r\)$ and its inverse function are:

<div>$$\int_0^{r'}{p_1(r)2\pi r\ dr}=\int_0^{r'}{\frac{e^{-r/d}}{d} \frac{1}{1-e^{-R_{max}/d}} dr}=\frac{-1}{1-e^{-R_{max}/d}}(e^{-r'/d -1}) \\
r=\log(1-\xi(1-e^{-R_{max}/d}))*-d$$</div>

Likewise, we can get the following formulas for the second exponential:

<div>$$p_2(r)=\frac{e^{-r/{3d}}}{2\pi dr}\frac{1}{3(1-e^{-R_{max}/{3d}})},\ r=\log(1-\xi(1-e^{-R_{max}/{3d}}))*{-3d}$$</div>

# Implementation Details

## Retrieve Correct Hit States from AiTraceProbe

<img src="/images/rlShaders/ai_false_probe_trace.jpg">

During surface shading computation, the shading context of sg is AI_CONTEXT_SURFACE, and I found `AiTraceProbe` would <span class="red">always return false when the probe ray hits an identical triangle to the one stored in previous sg.</span> To resolve this issue, at first I try to switch its shading context to AI_CONTEXT_DISPLACEMENT before each `AiTraceProbe`, then I found we could simply change `sg->fi` (the id of hit primitive) to UI_MAX to make it work properly.

In addition, about direct lighting evaluation, it would crash at `AiLightsPrepare` if we pass the returned AtShaderGlobals from `AiTraceProbe` directly. Thus we have to copy related information like P, Nf, Ng from in-scatter point to original sg before evaluating the light loop.

## Stackless Shading For Multiple Probe Hits

To shade thin geometry correctly, we need to shade multiple hits along the probing direction. Although we could use shader message passing API to carry the shading samples along the ray, but `AiTrace` would <span class="red">increase the number of call stacks and slow down the rendering.</span> Therefore, I use `AiTraceProbe` to retrieve the hit information and integrate direct/indirect illumination at each hit point.

## Reduce Incoming Radiance From Opposite Faces

Sometimes the energy from opposite faces would make the result too bright and need further reduction. For the consideration of surface cavity, I currently use $\cos{\theta/2}$ to fade out the incoming radiance. Where the $\theta$ is determined differently for front-facing ($x\_{i1}$) and back-facing ($x\_{i2}$) cases:

<img src="/images/rlShaders/bssrdf_cavity_affects_diffusion.png" style="box-shadow: none;">

<div>$$\theta = \left\{ \begin{array}{@{}ll@{}}
\arccos(\vec{n_o}\cdot\vec{n_i}), & \text{if } \vec{v}\cdot\vec{n_o} \ge 0 \\
\arccos(\vec{v}\cdot\vec{n_i}), & \text{otherwise}
\end{array}\right. , where\ \vec{v}\ is\ \frac{x_i-x_o}{\vert x_i-x_o\vert}$$</div>

The source code could be found [here](https://github.com/shihchinw/rlShaders/blob/master/src/rlSssND.cpp).

# Results

<a href="/images/rlShaders/rlSssNd_buddha.jpg" data-lightbox="results"><img src="/images/rlShaders/rlSssNd_buddha.jpg"></a>

<figure class="figure">
<a href="/images/rlShaders/rlSssNd_lucy.jpg" data-lightbox="results"><img src="/images/rlShaders/rlSssNd_lucy.jpg"></a>
<figcaption>The influence of cavity.</figcaption>
</figure>

<a href="/images/rlShaders/rlSssNd_dragon.jpg" data-lightbox="results"><img src="/images/rlShaders/rlSssNd_dragon.jpg" style="max-width: 70%"></a>


# References

1. Jensen, Henrik Wann, et al. <cite>["A practical model for subsurface light transport."](https://graphics.stanford.edu/papers/bssrdf/)</cite> SIGGRAPH 2001.
2. Donner, Craig Steven. <cite>[Towards realistic image synthesis of scattering materials.](http://www.cs.jhu.edu/~misha/Fall11/Donner.Thesis.pdf)</cite> PhD Thesis, University California, San Diego, 2006.
3. Habel, Ralf, Per H. Christensen, and Wojciech Jarosz. <cite>["Classical and Improved Diffusion Theory for Subsurface Scattering."](http://graphics.pixar.com/library/PhotonBeamDiffusion/index.html)</cite> Technical Report, 2013.
4. King, Alan, et al. <cite id="ref.4">["BSSRDF importance sampling."](https://sites.google.com/site/ckulla/home/bssrdf-importance-sampling)</cite> SIGGRAPH Talks, 2013.
5. Burley, Brent. <cite id="ref.5">["Extending the Disney BRDF to a BSDF with Integrated Subsurface Scattering."](http://blog.selfshadow.com/publications/s2015-shading-course/)</cite> SIGGRAPH Course Notes, 2015.
6. Christensen, Per H., and Brent Burley. <cite id="ref.6">["Approximate reﬂectance profiles for efficient subsurface scattering."](http://graphics.pixar.com/library/ApproxBSSRDF/index.html)</cite> Technical Report 15-04, Pixar, 2015.
