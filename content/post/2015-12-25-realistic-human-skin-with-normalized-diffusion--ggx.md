---
categories: []
coverImage: images/rlShaders/rlSkin_inf_head_uffizi.jpg
date: 2015-12-25T00:00:00Z
image: images/header/posts/2015-12-25-realistic-human-skin-with-normalized-diffusion--ggx.jpg
published: true
tags:
- Arnold
- BSSRDF
title: Realistic Human Skin with Normalized Diffusion & GGX
---

After implementing [BSSRDF importance sampling][BSSRDF MIS] for _Normalized Diffusion_ and _GGX BRDF_, I started wondering how realistic could the appearance of human skin be achieved with the combination of these two models. The whole development process is quite interesting, because it's a perfect combination of my two interests: programming and photography. With only one diffusion layer and two specular layers on top, I tried to find the way to make it as realistic as possible.

# Visual Components of Realistic Human Skin

Before even write any code, we need to know the key ingredients of the realistic human skin at first. Based on <cite>"The Appearance of Human Skin"</cite> by Takanori Igarashi et al. [[1](#ref.1)], the visual details observed by naked eye could be classified as:

* soft color diffusion from skin layers (i.e. epidermis, dermis and subcutis)
* glossy reflection caused by skin surface lipids
* fine wrinkles and hair
* wrinkles, creases, pores, freckles, spots

<figure>
<a href="https://en.wikipedia.org/wiki/Skin"><img src="/images/rendering/skin_layers.png"></a>
<figcaption>Skin structures. Image from <a href="https://en.wikipedia.org/wiki/Skin">wiki</a>.</figcaption>
</figure>

While the common layout of skin shader is modeled as (from top to bottom):

* skin surface lipids (thin oily layer)
* epidermis
* dermis
* subcutis

In this framework, the epidermis, dermis and subcutis are often referred to as shallow, mid and deep scattering. Among different renderers, the scattering layer is modeled by BSSRDF with various diffusion profile: [Quantized-Diffusion](http://www.eugenedeon.com/project/qd/) (Weta), Cubic/Gaussian ([Arnold](https://sites.google.com/site/ckulla/home/bssrdf-importance-sampling), [Cycles](https://www.blender.org/manual/render/cycles/nodes/shaders.html?#subsurface-scattering)), [Normalized Diffusion](http://blog.selfshadow.com/publications/s2015-shading-course/) ([RenderMan](http://renderman.pixar.com/resources/current/RenderMan/PxrSkin.html), Disney), and so on.

Other diffuse details, like wrinkles, pores, freckles, could be handled by regular approaches with color texture and normal/bump/displacement mapping.

The thin oily layer on top is an interesting one, its specular reflection is commonly modeled by physically based microfacet BRDF [[2](#ref.2), [3](#ref.3), [4](#ref.4), [5](#ref.5)] and it is sometimes decomposed into __sheen__ and __specular__ components in recent skin shader implementations. Now, the curious thing is why do we need two specular components to represent realistic reflection?

To my best knowledge, two specular lobes might come from the astonishing work done by Graham et al. [[2](#ref.2)]. They found the specular reflection from their measured data could be approximated well by <span class="orange">__two lobes__ of a _Beckmann_ distribution</span> instead of one. And this principle is also adopted in another amazing real time domo by von der Pahlen et al. [[3](#ref.3)].

# Layer Combination

<figure class="figure">
<a href="/images/rlShaders/rlSkin_aov.jpg" data-lightbox="results"><img src="/images/rlShaders/rlSkin_aov.jpg"></a>
<figcaption>Skin Shader AOVs</figcaption>
</figure>

In my skin shading experiment, two specular lobes are modeled with GGX BRDF with different roughness, while the sub-surface scattering is computed with the Normalized Diffusion profile. For the combination of these shading layers, the naïve way is simply to accumulate each shading result, but this would look a little _glow_ near the silhouettes [[4](#ref.4)]. The other way is to use the out-scattering Fresnel terms of given viewing direction to weight each layers, but this would make the silhouettes too dark [[6](#ref.6)].

Another more accurate way is to compute the weight for SSS layer with the energy which is not reflected from top oily layer [[2](#ref.2), [4](#ref.4), [5](#ref.5)]. In other words, it uses one minus the average hemispherical diffuse reflectance as weights for linear interpolation:

<div>$$\rho_{dt}(x,\vec{\omega_i})=1-\int_{2\pi}{f_r(x,\vec{\omega_o},\vec{\omega_i})(\vec{\omega_i}\cdot\vec{n}) d\vec{\omega_o}}$$</div>

where $f_r(x,\vec{\omega_o},\vec{\omega_i})$ is the microfacet BRDF

Although the average diffuse reflectance could be done at pre-processing stage (e.g. the node_initialize section), yet it would still affect the user experience of parameter tweaking in IPR mode. And just inspired by alSurface [[6](#ref.6)], I extract $F_t(\eta, \vec{\omega_i})$ out from the BSSRDF:

$$S_d(x_i,\vec{\omega_i};x_o,\vec{\omega_o})=\frac{1}{\pi}F_t(\eta,\vec{\omega_i})R_d(\vert x_i-x_o\vert)F_t(\eta,\vec{\omega_o})$$

and compute its average from GGX BRDF as blending weights:

$$sssWeight=1−specularAvgFresnel∗(1−sheenAvgFresnel)$$

With such layer combination, the visual difference between one and two glossy lobes is:

<div class="twentytwenty-container">
<img alt="One Lobe" src="/images/rlShaders/rlSkin_glossy_lobe1.jpg" />
<img alt="Two Lobes" src="/images/rlShaders/rlSkin_glossy_lobe2.jpg" />
</div>

and the influence of avgFresnel weighting is near the silhouette of cheek and nose:

<div class="twentytwenty-container">
<img alt="w/o avgFresnel" src="/images/rlShaders/rlSkin_glossy_lobe2_no_fresnel.jpg" />
<img alt="w/- avgFresnel" src="/images/rlShaders/rlSkin_glossy_lobe2.jpg" />
</div>

# Microstructure Details

<figure style="margin-top: 20px;">
<img src="/images/rendering/skin_microstructure.jpg">
<figcaption>Image form Graham et al. [2]</figcaption>
</figure>

As mentioned by Graham et al. [[2](#ref.2)], current face scanning techniques only provide mesostructure details like wrinkles, pores and creases. Hence they proposed a texture synthesis method to compute the high-resolution texture of microstructure bumps for more realistic reflection in close-ups.

In contrast, instead of using high-res synthesized texture, Jimenez and von der Pahlen [[3](#ref.2)] has demonstrated that the microstructure can be approximated with simple sinusoidal noise in real time context.

Since the storage of the microstructure texture is quite large, the micro-displacement map from [Digital Emily 2](http://gl.ict.usc.edu/Research/DigitalEmily2/) is about 298MB (and its tx file consumes 1.1GB!). Thus, I tried to follow the concept of second approach to blend a solid fractal noise with bump map texture to break the specular highlight:

<div class="twentytwenty-container">
<img alt="w/o noise" src="/images/rlShaders/rlSkin_glossy_lobe2.jpg" />
<img alt="w/- noise" src="/images/rlShaders/rlSkin_glossy_lobe2_noise.jpg" />
</div>

# Fading Out Diffusion by Surface Cavities

<img src="/images/rlShaders/bssrdf_cavity_affects_diffusion.png" style="box-shadow: none;">

The formula in my previous implementation was incorrect for back-facing diffusion. I've tweaked the formula a little bit, for back-facing diffusion, I use $\cos{\theta}$ as the weight for fadeout instead of $\cos{\theta/2}$. Where the $\theta$ is computed as:

<div>$$\theta = \left\{ \begin{array}{@{}ll@{}}
\arccos(\vec{n_o}\cdot\vec{n_i}), & \text{if } \vec{v}\cdot\vec{n_o} \ge 0 \\
\arccos(\vec{v}\cdot\vec{n_i}), & \text{otherwise}
\end{array}\right. , where\ \vec{v}\ is\ \frac{x_i-x_o}{\vert x_i-x_o\vert}$$</div>

With the consideration of surface cavities, it would reduce the diffusion amount from the opposite faces at front:

<div class="twentytwenty-container">
<img alt="w/o cavity" src="/images/rlShaders/rlSkin_cavity_off.jpg" />
<img alt="w/- cavity" src="/images/rlShaders/rlSkin_cavity_on.jpg" />
</div>

# Results

<a href="/images/rlShaders/rlSkin_inf_head_uffizi.jpg" data-lightbox="results"><img src="/images/rlShaders/rlSkin_inf_head_uffizi.jpg"></a>
<a href="/images/rlShaders/rlSkin_inf_head_stpeters.jpg" data-lightbox="results"><img src="/images/rlShaders/rlSkin_inf_head_stpeters.jpg"></a>
<a href="/images/rlShaders/rlSkin_inf_head_grace.jpg" data-lightbox="results"><img src="/images/rlShaders/rlSkin_inf_head_grace.jpg"></a>

<iframe src="https://player.vimeo.com/video/149999712" width="500" height="281" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>

As usual, the source code could be found [here](https://github.com/shihchinw/rlShaders/blob/master/src/rlSkin.cpp).

Graphics data sources:

* Model/textures: [Infinite Realistic Head Scan](http://ir-ltd.net/#cbp=http://ir-ltd.net/portfolio/infinite/)
* Light probes: [High-Resolution Light Probe Image Gallery](http://gl.ict.usc.edu/Data/HighResProbes/) from USC ICT

# Limitations of My Current Implementation

If the scatter distance is smaller than the edge length of hit triangle, the probe samples along U, V axis would totally lost, which would cause following artifact:

![Sample artifact](/images/rlShaders/rlSkin_artifact.jpg)

I've tried to increase the probability of the probing along normal direction for this case, but it still can't eliminate such artifact completely. Because at the surface point with low geometry curvature within a range of max scattering radius, it would still potentially waste most samples along U, V axis.

The other thing is about the scattering distance control. The normalized diffusion is _integrated to one for any positive distance $d$_, and current implementation distributes energy of each color channel among the given `scatterDist` components respectively.

The scattering energy is decayed from hit point, due to normalization, the farther distance it scatters the less energy it contributes near the hit point. In other words, to get more reddish results, we need to make the `scatterDist.r` relatively smaller than `scatterDist.g` and `scatterDist.b`. This might be a little counterintuitive to users. (This is a problem of my current implementation, not related to the normalized diffusion model itself. It seems that the parameters used in the course note [[7](#ref.7)] don't have such problem)

The last thing is about _energy conservation_ of layer combination, the layer blending so far is an ad-hoc solution. The new layer framework proposed by Jakob et al. [[8](#ref.8)] is an interesting direction for further studying. Besides, my GGX BRDF implementation would lost energy with high roughness values, this might be resolved with a new technique from Heitz et al. [[9](#ref.9)] which handles the multiple-scattering case within microfacet model.

# References

1. Igarashi, Takanori, Ko Nishino, and Shree K. Nayar. <cite id="ref.1">["The appearance of human skin: A survey."](http://www1.cs.columbia.edu/CAVE/publications/pdfs/Igarashi_CUTR05.pdf)</cite> Technical Report. 2005.
2. Graham, Paul, et al. <cite id="ref.2">["Measurement‐Based Synthesis of Facial Microgeometry."](http://gl.ict.usc.edu/Research/Microgeometry/)</cite> Eurographics. 2013.
3. Jimenez, Jorge. and Javier von der Pahlen. <cite id="ref.3">["Next Generation Character Rendering"](http://www.iryoku.com/stare-into-the-future)</cite> [[demo](https://www.youtube.com/watch?v=l6R6N4Vy0nE&feature=youtu.be)]. GDC'13.
4. d'Eon, Eugene, and David Luebke. <cite id="ref.4">["Advanced Techniques for Realistic Real-Time Skin Rendering".](http://http.developer.nvidia.com/GPUGems3/gpugems3_ch14.html)</cite> GPU Gems 3 Ch. 14. 2007.
5. Donner, Craig, and Henrik Wann Jensen. <cite id="ref.5">["Light diffusion in multi-layered translucent materials."](http://www.cs.virginia.edu/~jdl/bib/appearance/subsurface/donner05.pdf)</cite> SIGGRAPH. 2005.
6. Langlands, Anders. <cite id="ref.6">["Physically Based Shader Design in Arnold."](http://blog.selfshadow.com/publications/s2014-shading-course/)</cite> SIGGRAPH Course Notes. 2014
7. Burley, Brent. <cite id="ref.7">["Extending the Disney BRDF to a BSDF with Integrated Subsurface Scattering."](http://blog.selfshadow.com/publications/s2015-shading-course/)</cite> SIGGRAPH Course Notes, 2015.
8. Jakob, Wenzel. <cite id="ref.8">["layerlab: A computational toolbox for layered materials."](http://blog.selfshadow.com/publications/s2015-shading-course/)</cite> SIGGRAPH Course Notes, 2015.
9. Heitz, Eric, Johannes Hanika, Eugene d’Eon, and Carsten Dachsbacher. <cite id="ref.9">["Multiple-Scattering Microfacet BSDFs with the Smith Model."](https://eheitzresearch.wordpress.com/240-2/)</cite> Technical Report. 2015.

[BSSRDF MIS]:{{< relref "2015-10-27-bssrdf-importance-sampling-of-normalized-diffusion.md" >}}
