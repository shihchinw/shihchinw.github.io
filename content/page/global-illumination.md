---
title: Global Illumination
description: The adventure of Light Transport
published: true
---

# General Resources

* [Equation: Quantifying Light = Great 3-D Animation](http://www.wired.com/2010/07/st_equation_3danimation/) @WIRED
* <cite>The State of Rendering</cite> @fxguide. [Part1](http://www.fxguide.com/featured/the-state-of-rendering/), [Part2](http://www.fxguide.com/featured/the-state-of-rendering-part-2/).
    - Comprehensive introduction for current state of rendering research.
* [ompf2](http://ompf2.com)
    * Wonderful forum about ray tracing implementation and researching.

# GI in Production
Great materials for artists, TDs and R&D engineers.

* <cite>[The Comprehensive PBR Guide](https://www.allegorithmic.com/pbr-guide)</cite> Vol 1 & 2, from allegorithmic.
* <cite>[Global Illumination Across Industries](http://cgg.mff.cuni.cz/~jaroslav/gicourse2010/)</cite>. SIG'10.
* <cite>[Raytracers and workflow: a production perspective](http://dl.acm.org/authorize?6945594)</cite>. SIG'14.
* <cite>[The Quest for the Ray Tracing API](https://sites.google.com/site/raytracingapi)</cite>. SIG'16.
* <cite>[Path Tracing in Production](https://jo.dreggn.org/path-tracing-in-production/)</cite>. SIG'17.

# Textbooks

* <cite>[Physically Based Rendering 3/e](http://www.pbrt.org/contents.php)</cite>. Matt Pharr, Wenzel Jakob and Greg Humphreys.
* <cite>[Advanced Global Illumination](http://sites.edm.uhasselt.be/agibook/)</cite>. Philip Dutre, Kavita Bala and Phillips Bekaert.
* <cite>[Realistic Image Synthesis Using Photon Mapping](http://graphics.ucsd.edu/~henrik/papers/book/)</cite>. Henrik Wann Jensen.

# Physically-Based Shading

* <cite>Practical Physically Based Shading in Film and Game Production</cite>: [SIG'10](http://renderwonk.com/publications/s2010-shading-course/), [SIG'12](http://blog.selfshadow.com/publications/s2012-shading-course/)
* <cite>Physically Based Shading in Theory and Practice</cite>:
    [SIG'13](http://blog.selfshadow.com/publications/s2013-shading-course/), [SIG'14](http://blog.selfshadow.com/publications/s2014-shading-course/), [SIG'15](http://blog.selfshadow.com/publications/s2015-shading-course/), [SIG'16](http://blog.selfshadow.com/publications/s2016-shading-course/), [SIG'17](http://blog.selfshadow.com/publications/s2017-shading-course/)

# Light Transport

* <cite>[Robust Monte Carlo Methods for Light Transport Simulation][Eric Veach]</cite>. Eric Veach. 1997.
* <cite>[Optimizing Realistic Rendering with Many-Light Methods](http://cgg.mff.cuni.cz/~jaroslav/papers/mlcourse2012/)</cite>. SIG'12.
* <cite>[Ray Tracing is the Future and ever will be](https://sites.google.com/site/raytracingcourse/)</cite>. SIG'13.
* <cite>[Recent Advances in Light Transport Simulation: Theory & Practice](http://cgg.mff.cuni.cz/~jaroslav/papers/2013-ltscourse/)</cite>. SIG'13.
* <cite>[Recent Advances in Light Transport Simulation: Some Theory and a Lot of Practice](http://cgg.mff.cuni.cz/~jaroslav/papers/2014-ltscourse/)</cite>. SIG'14.
* <cite>[Path Integral Methods for Light Transport Simulation: Theory & Practice](http://cgg.mff.cuni.cz/~jaroslav/papers/2014-ltstutorial/)</cite>. EG'14.

## Monte Carlo Sampling & Analysis

* <cite>[Monte Carlo Ray Tracing](http://www.cs.rutgers.edu/~decarlo/readings/mcrt-sg03c.pdf)</cite>. SIG'03.
* <cite>[Frequency Analysis of Light Transport](https://belcour.github.io/blog/siggraph-2016-course.html)</cite>. SIG'16.
* <cite>[Fourier Analysis of Numerical Integration in Monte Carlo Rendering: Theory and Practice](https://www.cs.dartmouth.edu/~wjarosz/publications/subr16fourier.html)</cite>. SIG'16.
* <cite>[Blue-noise Dithered Sampling](https://www.solidangle.com/research/dither_abstract.pdf)</cite>. SIG'16.
* <cite>[Area-Preserving Parameterizations for Spherical Ellipses](https://www.solidangle.com/research/egsr2017_spherical_ellipse.pdf)</cite>. EGSR'17.

## Importance Sampling

* <cite>[Robust Monte Carlo Methods for Light Transport Simulation][Eric Veach]</cite>, Ch 2, 9. Eric Veach. 1997.
* <cite>[IS for Production Rendering](https://sites.google.com/site/isrendering/home)</cite>. SIG'10.
* <cite>[Importance Sampling of Area Lights in Participating Media](https://sites.google.com/site/ckulla/home/importance-sampling-of-area-lights-in-participating-media)</cite>. SIG'11 Talks.
* <cite>[BSSRDF Importance Sampling](https://sites.google.com/site/ckulla/home/bssrdf-importance-sampling)</cite>. SIG'13 Talks.

## Photon Mapping

* <cite>[Realistic Image Synthesis Using Photon Mapping](http://graphics.ucsd.edu/~henrik/papers/book/)</cite>. Henrik Wann Jensen.
* <cite>[State of the Art in Photon Density Estimation](http://users-cs.au.dk/toshiya/starpm2012/)</cite>. SIG'12.

## Volume Rendering

* <cite>[Importance Sampling Techniques for Path Tracing in Participating Media](https://www.solidangle.com/research/egsr2012_volume.pdf)</cite>. EGSR'12.
* <cite>Production Volume Rendering</cite>: [SIG'11](http://magnuswrenninge.com/productionvolumerendering), [SIG'17](http://graphics.pixar.com/library/ProductionVolumeRendering/index.html).

# Programming
Learning from coding!

* [PBRT v3](https://github.com/mmp/pbrt-v3/)
* [Mitsuba](https://www.mitsuba-renderer.org)
* [Nori 2](http://wjakob.github.io/nori/)
* [Tungsten](https://benedikt-bitterli.me/tungsten.html)

## Ray Tracing Engines
* [OptiX](https://developer.nvidia.com/optix) from NVidia
* [Embree](http://embree.github.io) from Intel

## Data Structure & Algorithms
* <cite>[Efficient Sorting and Searching in Rendering Algorithms](http://www.researchgate.net/publication/261624776_EG_2014_tutorial_Efficient_Sorting_and_Searching_in_Rendering_Algorithms_-_slides)</cite>, EG'14.


[Eric Veach]: https://graphics.stanford.edu/papers/veach_thesis/
