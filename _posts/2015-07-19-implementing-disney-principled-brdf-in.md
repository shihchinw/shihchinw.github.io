---
layout: post 
title: Implementing Disney Principled BRDF in Arnold
date: '2015-07-19T16:05:00.000+08:00'
categories: []
tags: [Arnold, BRDF, Importance Sampling]
published: True
cover-img: "images/rlShaders/rlDisney_bunny_row.jpg"
header-img: "images/header/posts/2015-07-19-implementing-disney-principled-brdf-in.jpg"

---

Disney "principled" BRDF [[1](#ref.1)] is a very interesting shading model. It can achieve various physically plausible appearances with only one color parameter and ten scalar parameters! This shading model is designed not only to catch most of the measured data from [MERL](http://www.merl.com/brdf/), but also to follow the principles below:

> 1. __Intuitive__ rather than physical.
> 2. As __few parameters__ as possible.
> 3. Parameters are __zero to one__ over their plausible range.
> 4. Parameters are __allowed to be pushed beyond__ their plausible range where it makes sense.
> 5. All combinations of parameters should be as __robust and plausible__ as possible.

These art-directable features make artists achieve the desired shading results more easily, it is one of the ultimate goals for software engineering in animation studio. Since it has been implemented within various frameworks like [Unreal Engine 4](http://blog.selfshadow.com/publications/s2013-shading-course/), [RenderMan](http://renderman.pixar.com/resources/current/RenderMan/PxrDisney.html), MPC's preview system [Dekko](http://on-demand-gtc.gputechconf.com/gtcnew/on-demand-gtc.php?searchByKeyword=mpc&searchItems=&sessionTopic=&sessionEvent=2&sessionYear=2015&sessionFormat=&submit=&select=), etc. So, let's make one for Arnold. :)

# Importance Sampling

According to the source code from BRDF explorer, the shading components can be separated into two parts:

* diffuse
    * base diffuse
    * Hanrahan-Krueger subsurface
* specular
    * primary specular (anisotropic GTR2)
    * clearcoat (GTR1)
    * sheen

## Diffuse

From BRDF explorer, we can see that the diffuse component is dominated by the $\cos{\theta_l}$ factor. Therefore the diffuse part could be sampled with the PDF proportional to cosine-weighted solid angle:
![diffuse sample distribution](/images/rlShaders/rlDisney_is_diffuse.jpg)

## Specular

Sheen is relatively small to primary specular (GTR2) and clearcoat (GTR1) and it is ignored in the importance sampling process in current implementation. The importance sampling approach for GTR is pretty much like we did for GGX previously, except for the anisotropic case which is a little complicated (I've not successfully derived the formula yet). Fortunately, Burley has kindly provided the details in the course notes [[1](#ref.1)], and we could simply follow those formulas to generate samples:

$$\begin{eqnarray}
aspect&=&\sqrt{1-0.9*anisotropic} \nonumber \\
\alpha_x&=&\frac{roughness^2}{aspect},\ \alpha_y=roughness^2*aspect \nonumber \\
\end{eqnarray}$$

for primary anisotropic specular:

$$\begin{eqnarray}
D_{GTR_2aniso}&=&\frac{1}{\pi}\frac{1}{\alpha_x\alpha_y}\frac{1}{(\sin^2{\theta_h}(\frac{\cos^2{\phi_h}}{\alpha_x^2}+\frac{\sin^2{\phi_h}}{\alpha_y^2})+\cos^2{\theta_h})^2} \nonumber \\
h'&=&\sqrt{\frac{\xi_2}{1-\xi_2}}[\alpha_x\cos(2\pi\xi_1)\vec{u}+\alpha_y\sin(2\pi\xi_1)\vec{v}]+\vec{n},\ h=\frac{h'}{|h'|}
\end{eqnarray}$$

for clearcoat:

$$\begin{eqnarray}
D_{GTR_1}&=&\frac{\alpha^2-1}{\pi\log\alpha^2}\frac{1}{(1+(\alpha^2-1)\cos^2\theta_h)} \nonumber \\
\cos\theta_h&=&\sqrt{\frac{1-(\alpha^2)^{1-\xi_2}}{1-\alpha^2}},\ \phi_h=2\pi\xi_1
\end{eqnarray}$$

The microfacet normal h is generated according to $D(\theta_h)\cos\theta_h$ which is already normalized over the hemisphere. Therefore, I tried to use the coefficents of $D_{GTR_2}, D_{GTR_1}$ as weights for lobe selection and PDF composition:

{% highlight cpp linenos %}
AtVector    sampleSpecularDirection(float rx, float ry) const
{
    // Select lobe by the relative weights.
    // Sample the microfacet normal first, then compute the reflect direction.
    AtVector M;

    float gtr2Weight = 1.0f / (mClearcoat + 1.0f);

    if (rx < gtr2Weight) {
        rx /= gtr2Weight;
        M = sampleGTR2AnisoDirection(rx, ry);
    } else {
        rx = (rx - gtr2Weight) / (1.0f - gtr2Weight);
        M = sampleGTR1Direction(rx, ry);
    }

    if (AiV3Dot(mAxisN, M) < 0.0f) {
        return AI_V3_ZERO;
    }

    return getReflectDirection(mViewDir, M);
}

float   evalSpecularPdf(const AtVector &i) const
{
    auto m = AiV3Normalize(i + mViewDir);
    auto IdotM = ABS(AiV3Dot(i, m));
    auto MdotN = AiV3Dot(m, mAxisN);

    if (MdotN < 0.0f) {
        return 0.0f;
    }

    float MdotN2 = SQR(MdotN);

    auto clearcoatWeight = mClearcoat / (mClearcoat + 1.0f);
    auto D = LERP(clearcoatWeight, D_GTR2Aniso(m, MdotN2), D_GTR1(m, MdotN2));
    return D * ABS(MdotN) * 0.25f / IdotM;
}
{% endhighlight %}

<figure class="figure">
<img src="/images/rlShaders/rlDisney_is_specular_a.4_theta_10d.jpg">
<figcaption>roughness=0.4, $\theta_o=10^\circ$</figcaption></figure>

<figure class="figure">
<img src="/images/rlShaders/rlDisney_is_specular_a.4.jpg">
<figcaption>roughness=0.4, $\theta_o=45^\circ$</figcaption></figure>

<figure class="figure">
<img src="/images/rlShaders/rlDisney_is_specular_a.4_theta_80d.jpg">
<figcaption>roughness=0.4, $\theta_o=80^\circ$</figcaption></figure>

<figure class="figure">
<img src="/images/rlShaders/rlDisney_is_specular_a.8.jpg">
<figcaption>roughness=0.8, $\theta_o=45^\circ$</figcaption></figure>

The <span class="red">red dots</span> at bottom represent the invalid sample whose $\theta_m$ is greater than 90 degrees (i.e. $m \cdot n < 0$). It might be able to get improved by importance sampling the <span class="orange">__visible__</span> microfacet normals presented in Heitz and d'Eon [[3](#ref.3)].

# Removing Fireflies

Current shading model is not energy conserved, sometimes the specular part would cause fireflies in indirect diffuse pass. The noise level can be reduced by using metallic parameter to blend the diffuse and GTR parts, but it would lost the clearcoating effect for non-metal material. Another possible approach is to skip specular evaluation for indirect ray samples (i.e. AI_RAY_DIFFUSE/AI_RAY_GLOSSY) like tutorial does with the price of missing reflection of metallic material.

![indirect lighting](/images/rlShaders/rlDisney_indirect.jpg)

After few trial and errors, I found the most intuitive way is to use extra weights to scale the diffuse/specular radiance for indirect ray samples:

{% highlight cpp linenos %}
if (sg->Rt & (AI_RAY_DIFFUSE | AI_RAY_GLOSSY)) {
    diffuse *= AiShaderEvalParamFlt(p_indirect_diffuse);
    specular *= AiShaderEvalParamFlt(p_indirect_specular);
}
{% endhighlight %}

<a href="/images/rlShaders/rlDisney_indirect_scale_control.jpg" data-lightbox="results"><img src="/images/rlShaders/rlDisney_indirect_scale_control.jpg"></a>

# Results

<figure class="figure">
<a href="/images/rlShaders/rlDisney_metallic.jpg" data-lightbox="results"><img src="/images/rlShaders/rlDisney_metallic.jpg"></a>
<figcaption>metallic 0 to 1</figcaption></figure>

<figure class="figure">
<a href="/images/rlShaders/rlDisney_roughness.jpg" data-lightbox="results"><img src="/images/rlShaders/rlDisney_roughness.jpg"></a> 
<figcaption>roughness 0 to 1</figcaption></figure>

<figure class="figure">
<a href="/images/rlShaders/rlDisney_test1.jpg" data-lightbox="results"><img src="/images/rlShaders/rlDisney_test1.jpg"></a>
<a href="/images/rlShaders/rlDisney_test2.jpg" data-lightbox="results"><img src="/images/rlShaders/rlDisney_test2.jpg"></a></figure>

<a href="/images/rlShaders/rlDisney_bunny_row.jpg" data-lightbox="results"><img src="/images/rlShaders/rlDisney_bunny_row.jpg"></a>
<a href="/images/rlShaders/rlDisney_lucy.jpg" data-lightbox="results"><img src="/images/rlShaders/rlDisney_lucy.jpg"></a>

The source code could be found [here](https://github.com/shihchinw/rlShaders/blob/master/src/rlDisney.cpp). If you have any idea or find any bug, please let me know in the comments.

# References
1. Burley, Brent. <cite id="ref.1">"[Physically Shading in Disney. 3/e.](http://blog.selfshadow.com/publications/s2012-shading-course/burley/s2012_pbs_disney_brdf_notes_v3.pdf)"</cite> SIGGRAPH 2012 [Course Notes](http://blog.selfshadow.com/publications/s2012-shading-course/). [[source code](https://github.com/wdas/brdf/blob/master/src/brdfs/disney.brdf)]
2. Walter, Bruce, et al. <cite>["Microfacet models for refraction through rough surfaces."](http://www.cs.cornell.edu/~srm/publications/EGSR07-btdf.html)</cite> EGSR'07.
3. Heitz, Eric, and Eugene d'Eon. <cite id="ref.3">["Importance Sampling Microfacet‐Based BSDFs using the Distribution of Visible Normals."](https://hal.inria.fr/hal-00996995)</cite> EGSR'14. [[slides](https://hal.inria.fr/hal-00996995v2/file/slides.pdf)]
