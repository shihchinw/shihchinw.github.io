---
layout: post 
title: Using Min Pixel Width with Care in Arnold
categories: []
tags: [Arnold]
published: True
cover-img: "images/arnold/mtoa_hair_mpw_opaque_cmp_1k.jpg"

---

Keeping objects opaque makes Arnold render much faster. When we render a bunch of hairs, `min pixel width` is useful to make the hair look softer. However, if the `min pixel width` is not equal to zero, the opacity of hair ribbon would change subtly, and those ribbons are not always opaque anymore! Furthermore, according to the ribbon thickness measured in screen space at render time, we might surprisedly get different render results. Thus, special care should be taken to tweak this parameter under various circumstances.

<figure class="figure">
<a href="/images/arnold/mtoa_hair_mpw_opaque_dist.jpg" data-lightbox="posts">
<img src="/images/arnold/mtoa_hair_mpw_opaque_dist.jpg"></a>
<figcaption class="figure-caption">Opaque mode is switched based on the thickness measured in screen space.<br>Tiny distance changes would make different results.</figcaption></figure>

# Min Pixel Width & Opaque

`Min pixel width` is used to resolve aliasing problem by enlarging the ribbon thickness and decreasing its shading opacity at the same time.

<img src="/images/arnold/mtoa_opacity_with_min_pixel_width.jpg">
<figcaption class="figure-caption">Min pixel width enlarges hair thickness and decrease opacity at the same time.</figcaption>

The opaque mode of ribbon from Maya hair would be <span class="red">__switched off__</span> when:

1. HairSystem:opaque is off, or
2. HairSystem:min_pixel_width takes effect (i.e. the raster size of ribbon is smaller than `min pixel width`)

<figure>
<img src="/images/arnold/mtoa_curve_opaque_switch.jpg">
<figcaption>Download: <a href="https://drive.google.com/file/d/0BzMAAU0MqqtQYjRmbVR0ZEJKR3c/view?usp=sharing">ai_curve_mpw_test.ma</a></figcaption></figure>

It's not recommended to enlarge thickness of opaque ribbons by `mix pixel width` directly, because this would not keep ribbons opaque all the time. Besides, there is a potential pitfall which would increase the "depth complexity" of each ray. Since when a ray traces toward semi-transparent objects, it has to trace all the way through the scene to accumulate the shading color <span class="blue">until the resultant opacity adds to one or the trace depth exceeds the transparent depth.</span>

# Depth Complexity

Suppose there are three semi-transparent planes covering a single pixel and AA is set to 4 (16 camera rays per pixel). If we use _"depth complexity"_ to describe the potential traced depth of each ray. The sample pattern within a pixel might look like this:

<figure>
<img src="/images/arnold/transparent_depth_0-3.png">
<figcaption>Depth complexity = {gray: 0, blue: 1, green: 2, red: 3}</figcaption></figure>

If we set `min pixel width` to one, each ribbon would always cover one pixel width at least. Then the depth complexity for each sample becomes three. In this situation, the worst case is all rays are tracing through three planes to compute the resultant shading color.

<img src="/images/arnold/transparent_depth_3.png">

For production rendering, there are hundreds of thousands ribbons at least. Imaging we have plenty of nearly transparent ribbons covering a single pixel. Most rays would exhaust their transparent depth, and need more time to render each frame. The most surprising case is presented by Jesse Andrewartha in _"Establishing best practices in raytracing at Sony Pictures Imageworks, p.18"_ [[1](#ref.1)]: <span class="red">__5% pixels cost 99% render time!!__</span>

# Test Results

<a href="/images/arnold/mtoa_hair_mpw_opaque_cmp.jpg" data-lightbox="results">
<img src="/images/arnold/mtoa_hair_mpw_opaque_cmp.jpg"></a>

#### Image Size: 512x512

| Opaque | MPW |Shadow Rays | Diffuse Rays | Rays/Pixel | Render Time|
| :--:   | :-: | --------:  | -------:     | -----:     | :-:        |
| on     | 0.0 | 7402637    | 4775669      | 67.85      | 0:19       | 
| on     | 0.2 | <span class="red">__119529920__</span>  | 26825142     | <span class="red">__549.15__</span>  | <span class="red">__9:23__</span>       | 
| off    | 0.0 | 37367418   | 13250617     | 205.74     | 2:55       |

<figure>
<a href="/images/arnold/mtoa_hair_mpw_opaque_cmp_1k.jpg" data-lightbox="results"><img src="/images/arnold/mtoa_hair_mpw_opaque_cmp_1k.jpg"></a>
<figcaption>Download: <a href="https://drive.google.com/file/d/0BzMAAU0MqqtQQUxTSDhNbzBIdjg/view?usp=sharing">ai_hair_mpw_test.ma</a></figcaption></figure>

#### Image Size: 1K

| Opaque | MPW |Shadow Rays | Diffuse Rays | Rays/Pixel | Render Time|
| :--:   | :-: | --------:  | -------:     | -----:     | :-:        |
| on     | 0.0 | 31580697   | 20379657     | 70.8       | 1:17       |
| off    | 0.0 | 159407301  | 56561199     | 217.87     | 13:09      |
| on     | 0.2 | 179140421  | 62032101     | 240.41     | 14:31      |
| on     | 0.5 | <span class="red">__715261105__</span>  | 139611571    | <span class="red">__790.75__</span>     | <span class="red">__52:47__</span>      |

# Conclusion

For hair/fur rendering, it's always a good exercise to expose hair density and width control to lighting artists. In addition, only use `min pixel width` for softness/anti-aliasing; hair thickness should be configured from <span class="green">__HairSystem:width__</span> instead of `min pixel width`.

In the case of thick hair whose width is larger than the size of several pixels. If we want to make those thick hair translucent, it's better to turn off the <span class="green">__HairSystem:opaque__</span> and adjust the opacity from shader. Use `min pixel width` to switch opacity mode might not always get consistent results and have a potential pitfall which would increase render time dramatically.

# References

1. Andrewartha, Jesse, Søren Ragsdale, and Paul Beilby. <cite id="ref.1">["Raytracers and workflow: a production perspective."](http://dl.acm.org/authorize?6945594)</cite> SIGGRAPH 2014 Course Notes.
2. [Min pixel width](https://support.solidangle.com/display/AFMUG/Curves#Curves-Min.PixelWidth) from MtoA User Guide.
