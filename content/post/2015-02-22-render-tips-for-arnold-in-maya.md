---
categories: []
coverImage: images/arnold/mtoa_tips.png
date: 2015-02-22T16:39:00Z
image: images/arnold/mtoa_tips.png
published: true
tags:
- Arnold
- Maya
title: Render Tips for Arnold in Maya
---

All photo-realistic ray-tracers are trying to solve the [rendering equation](http://www.wired.com/2010/07/st_equation_3danimation/). Although they are all categorized as ray-tracer instead of rasterizer, the infrastructure of each implementations are quite different (i.e. uni-directional vs. bi-directional, pure path tracing vs. works with photon-map or [irradiance caching](http://cgg.mff.cuni.cz/~jaroslav/papers/2008-irradiance_caching_class/), etc).

Arnold is an _un-bias uni-directional path tracer_ with a remarkable importance sampling mechanism and great memory management to achieve amazing performance excellence. However, if we blindly increase the number of ray samples for every render scene, we would not always get a cleaner imagery. Which means that when we switch from one ray-tracer to another, we had better use different strategies to adjust render settings. Here is the summary about what I've learned from experience, hope it would help any Arnold user. :)
<!--more-->

# General

1. <span class="orange">__Always using tx!__</span> It can significantly reduce I/O time!
![tx](/images/arnold/mtoa_use_existing_tx.jpg)
2. Change log verbosity to `Warnings + Info`. This is very useful for render process diagnosis. ![mtoa_verbosity](/images/arnold/mtoa_log_verbosity.jpg)
3. Configure [file output](http://forums.cgsociety.org/showthread.php?t=1089478)
![mtoa output setting](/images/arnold/mtoa_output_setting.jpg)
  * Set compression format to `zips` and enable `half-precision`, `auto-crop` to save storage.
  * <del>Turn off `tiled`, since most compositors prefer scanline format.</del> This is only valid before Nuke 7. Reading tiled EXR in Nuke 9 is <span class="orange">[**25 times faster**](http://www.cgchannel.com/2014/10/the-foundry-details-new-features-of-nuke-9-nukex-9/)</span>. Besides, scanline EXR has [memory overhead](#FDBK1) when rendering many AOVs. So let's turn `tiled` **ON** now!
4. Check output log for any missing or broken textures.
  * Arnold will stop rendering process immediately when it fails to access target textures.
5. Use `aiUtility` to examine the problems of geometry such as:
    * over-smoothing (tessellation) mesh
        * The number of triangles would affect the render performance since ray tracer needs to load all the objects in the scene before tracing.
    * face penetration, discontinuous tangent/normal vectors, etc
        * Those problem would cause discontinuous shading artifacts.

# Linear Workflow
The best practice of photo-realistic rendering should be done with [linear workflow](http://filmicgames.com/archives/299). Maya supports color management after Maya 2015, but few things should be noticed. When we turn on the color management in Maya:
![Color Management in Maya](/images/maya/maya_color_management.png)

We need to set all parameters in `gamma correction` to __1.0__ and set display gamma to sRGB, Rec 709, etc:
![Gamma Correction in Render Setting](/images/arnold/mtoa_gamma_correction.png)
![Gamma Correction in Render View](/images/maya/maya_display_gamma.png)

The last thing is that MtoA doesn't support `color space` configuration in file node currently (v1.2.4.3).
![Color space control in file node](/images/arnold/mtoa_file_color_space.png)

We could use extra gamma nodes to convert images to linear space, but the recommended approach is to use following command to convert the color texture file to avoid run-time shading overhead:
<samp>maketx --colorconvert sRGB linear path-to-image</samp>

# Look-Dev

1. Ensure **"Energy Conservation"** (i.e. outgoing energy ≤ incoming energy)
  * In physically-based setup, the energy of each ray should not get higher after each bounce, or you might get [fireflies](https://support.solidangle.com/display/ARP/Fireflies) in your render images. There are [two approaches](https://support.solidangle.com/display/AFMUG/Standard#Standard-EnergyConservation) to ensure this:
    1. Make sure the sum of relevant parameters is less than one, or
    2. Turn on Fresnel for diffuse/specular/reflection/refraction
2. Avoid unnecessary transparent setting.
  * It might be helpful to use [set-override](https://support.solidangle.com/display/AFMUG/Override+Sets) to switch the `aiOpaque` attribute for LOD. (We can switch aiOpaque on for the objects which are far from camera.)
3. Use [`aiRaySwith`](https://support.solidangle.com/display/AFMUG/Ray+Switch) to simplify the shading of indirect bounce (the detail is beyond this post).
4. If possible, avoid blending multiple material shaders (e.g. aiStandard, aiSkin)
  * Try to compose the control parameters into textures and connect those textures to a single shader.
  * Since each material shader would spawn rays to compute a resultant color, thus the more material shaders we use in a single shading network, the more rays are fired for a resultant color computation, or
  * Try using [alLayer](https://bitbucket.org/anderslanglands/alshaders/wiki/alLayer) in alShaders [[1](#ref.1)] for complicated mixture materials.
5. In general case, try to use [specular](https://support.solidangle.com/display/AFMUG/Standard#Standard-SpecularityvsReflection) rather than reflection in aiStandard.
  * Unlike specular, reflection would not sample the lights!
6. Prefer [mesh light](https://support.solidangle.com/display/AFMUG/Mesh+Light#MeshLight-MeshLightvsEmission) over emission in aiStandard.

# Lighting

1. Before up-sampling to remove the noise, <span class="orange">examine each AOV for the source of noise first!!</span>
![mtoa AOV browser](/images/arnold/mtoa_aov_setting.jpg) ![mtoa AOVs](/images/arnold/mtoa_aovs.jpg)
2. Keep the number of **rays/pixel** in a reasonable range
    * As mentioned in [[2](#ref.2)], 2000~3000 rays/pixel is not uncommon in _"The Amazing Spider Man 2"_.
    * The rule-of-thumb is to strike a balance between AA and other ray type samples:
        * <span class="blue">AA ↑, Diffuse/Glossy/Refraction/SSS/Light ↓</span> or
        * <span class="blue">AA ↓, Diffuse/Glossy/Refraction/SSS/Light ↑</span>
    * <span class="blue">When motion blur is on, reset all sample number to 1 and increase AA sample from 5 progressively.</span>
        * Remember, the actual sample count is the **square of the value** of the sample attribute.
3. Check CPU & ray-count AOVs to examine the performance bottleneck.
I personally use [imf_disp](http://docs.autodesk.com/MENTALRAY/2015/CHS/mental-ray-help/files/manual/imf_disp.html) (provided by mental ray :D) to view EXR files. When we examine the image whose pixel values are higher than 1, we have to decrease the exposure value for better visual display. In addition, the status bar at bottom would show the pixel value pointed by the cursor.
<img src="/images/arnold/mtoa_ray_count.jpg" width="70%">
4. For the objects which need relatively high sample count compared to the rest of the scene, separate them into another render layer with extra high samples. (e.g. complex refractive geometry or rough specular object)
5. Avoid unnecessary ray depth.
  * Do we really need a second/third bounce diffuse/specular in outdoor scene? If there is no or tiny visual differences, it would be better to keep the ray depth as shallow as possible.
6. Use a dummy geometry (whose primary visibility is off) as bounce card.
7. Prune unnecessary lights which contribute low energy to the scene.
8. Hide unnecessary objects which has no impact to the render image [[4](#ref.4)].
  * It might be helpful to use a frustum to hide objects far away (i.e. visibility culling)
9. Use `min pixel width` with care for hair/fur rendering, it would change opaque mode subtly. [[more details]({% post_url 2015-09-24-using-min-pixel-width-with-care-in-arnold %})]

# Feedback

<blockquote id="FDBK1" class="feedback"><a href="http://www.sosoyan.com/">Sosoyan</a>
<time>November 28, 2015</time>

Not sure about to Turn off “Tiled”, because in this case raw buckets are being stored in the memory until the render finished. First you are getting waste of memory especially if you have many Aovs you can get around 1.5GB more than tiled, second while tiled is on you can view your unfinished render in progress for example with RV.

In terms of scanline thing, I assume this post is to old, because Nuke 9 is fine with tiled Exrs now.
</blockquote>

# References

1. <cite id="ref.1">[alShaders](http://anderslanglands.com/alshaders/index.html)</cite>. Wonderful 3rd party physically-based shaders developed by Anders Langlands.
2. <cite id="ref.2">[Raytracers and workflow: a production perspective](http://dl.acm.org/authorize?6945594)</cite>, SIGGRAPH'14 Course Note
  * There are many in-depth optimization tips for SPI Arnold (but some concepts are the same in Arnold). Highly recommended! A must-read document for Arnold users.
3. [VFX Tiger and Tips Using The Arnold Renderer](http://blog.animationmentor.com/vfx-tiger-and-tips-using-the-arnold-renderer/) by Animation Mentor
4. Repasky, Zachary, et al. <cite id="ref.4">[Large Scale Geometric Visibility Culling on Brave](http://graphics.pixar.com/library/VisibilityCulling/)</cite>. PIXAR Technical Report. 2013.
