---
categories: []
coverImage: images/rlShaders/rlGgx_lucy.jpg
date: 2015-06-28T15:20:00Z
image: images/header/posts/2015-06-28-implementing-ggx-brdf-in-arnold-with.jpg
published: true
tags:
- Arnold
- BRDF
- Importance Sampling
title: Implementing GGX BRDF in Arnold with Multiple Importance Sampling
---

Monte Carlo path tracing has a slow convergence rate close to \\(O(1/\sqrt n)\\), it needs to quadruple the number of samples to halve the error. Many mainstream production renderers have deployed multiple importance sampling (MIS) technique to reduce the variance of sampling result (i.e. the noise in image). If we want to make our custom material shader efficiently executed along with the entire render scene in Arnold, we shall definitely consider exploiting the power of MIS.

![MIS](/images/arnold/ai_mis_cmp.png)

# What is Multiple Importance Sampling (MIS)?
Suppose we want to solve the direct illumination equation:

<div>$$L_o(x,\vec{\omega_o}) = \int_\Omega L_i(x, \vec{\omega_i}) f_r(x, \vec{\omega_o},\vec{\omega_i}) \cos\theta_i\, d\omega_i$$</div>

Monte Carlo sampling uses random samples to compute this integral numerically. The idea of importance sampling is trying to generate the random samples proportional to some probability density function (PDF) which has similar shape of the integrand.

<figure class="figure">
<img src="/images/rendering/importance_sampling_concept.png">
<figcaption class="figure-caption">Image from Simon Premoze, <a href="https://sites.google.com/site/isrendering/Simon.Presentation.pdf?attredirects=0">Practical Importance Sampling and Variance Reduction</a>, SIGGRAPH Course Notes, 2010.</figcaption>
</figure>

In other words, the ideal PDF for the integrand of direct illumination equation would be proportional to the product of incoming radiance \\(L_i(x,\vec{\omega_i})\\) and BRDF \\(f_r(x,\vec{\omega_o}, \vec{\omega_i})\\). However, the product of these terms are too complicated to compute its PDF, but we might be able to find a suitable PDF for some of those terms individually. What we hope is to find a way to sample each term respectively and properly combine those samples to get a result with lower variance. This turns to the technique named __Multiple Importance Sampling__ (MIS) demonstrated by Veach and Guibas [[1](#ref.1)] in 1995:

<figure class="figure">
<a href="/images/rendering/mis.png" data-lightbox="post">
<img src="/images/rendering/mis.png"></a>
<figcaption class="figure-caption">Images from Veach and Guibas [1]</figcaption></figure>

With MIS, we use following formula to compute the result:

<div>$$\begin{gathered}
\kern{5em} \frac{1}{n_f}\sum_{i=1}^{n_f}\frac{f(X_i)g(X_i)w_f(X_i)}{p_f(X_i)}+\frac{1}{n_g}\sum_{j=1}^{n_g}\frac{f(Y_j)g(Y_j)w_g(Y_j)}{p_g(Y_j)}\kern{5em} (1)
\end{gathered}$$</div>

Where \\(n_k\\) is the number of samples taken from certain PDF \\(p_k\\). The weighting functions \\(w_k\\) takes all the different ways that a sample could have been generated, and \\(w_k\\) could be computed by _power heuristic_:
$$w_k(x)=\frac{(n_s p_s(x))^\beta}{\sum_i{(n_i p_i(x))^\beta}}$$

# The MIS API in Arnold

At each shading point, we want to compute the direct illumination and indirect illumination. In order to utilize MIS API in Arnold, we have to provide three functions:

1. <span style="color:#FF3399;">AtBRDFEval**Pdf**Func</span>
2. <span style="color:#009933">AtBRDFEval**Sample**Func</span>
3. <span style="color:#0099FF">AtBRDFEval**Brdf**Func</span>

For getting better understanding of the task of these functions. Recall that our goal is to use MIS to combine the samples generated from both PDFs of lights and BRDF. Let BRDF as \\(f(X)\\) and lighting function as \\(g(Y)\\) in equation (1), then the unknown entities to Arnold are:

* \\(\textcolor{#FF3399}{p_f}\\) (AtBRDFEvalPdfFunc): the PDF of BRDF
* \\(\textcolor{#009933}{X_i}\\) (AtBRDFEvalSampleFunc): the sample direction generated according to \\(\textcolor{#FF3399}{p_f}\\)
* \\(\textcolor{#0099FF}{f}\\) (AtBRDFEvalBrdfFunc): the BRDF evaulated with sampled incoming direction and fixed outgoing direction (i.e. -sg->Rd)

With these information, \\(w_f(X_i)\\) and \\(w_g(Y_j)\\) can be computed with _power heuristic_:

<div>$$\begin{aligned}
w_f(\textcolor{#009933}{X_i}) &= \frac{(n_f \textcolor{#FF3399}{p_f}(\textcolor{#009933}{X_i}))^\beta}{(n_f \textcolor{#FF3399}{p_f}(\textcolor{#009933}{X_i})+n_g p_g(\textcolor{#009933}{X_i}))^\beta} \\
w_g(Y_j) &= \frac{(n_g p_g(Y_j))^\beta}{(n_f \textcolor{#FF3399}{p_f}(Y_j)+n_g p_g(Y_j))^\beta}
\end{aligned}$$</div>

Now suppose we want to use GGX for glossy component, the shading structure would like this:

<pre class="prettyprint linenums lang-cpp">
GgxBrdf ggx_brdf;
GgxBrdfInit(&ggx_brdf, sg);

auto directGlossy = AI_RGB_BLACK;

AiLightsPrepare(sg);
while (AiLightsGetSample(sg)) {
  if (AiLightGetAffectSpecular(sg->Lp)) {
    directGlossy += AiEvaluateLightSample(sg, &ggx_brdf,
      ggxEvalSample, ggxEvalBrdf, ggxEvalPdf);
  }
}

auto indirectGlossy = sg->Rr > 0
  ? AI_RGB_BLACK
  : AiBRDFIntegrate(sg, &ggx_brdf,
      ggxEvalSample, ggxEvalBrdf, ggxEvalPdf, AI_RAY_GLOSSY);
</pre>

## BRDF Evaluation
The content of `ggxEvalBrdf` is the formula of microfacet BRDF:

$$f_r(i,o,n)=\frac{F(i, h_r)G(i, o, h_r)D(h_r)}{4|i⋅n||o⋅n|}$$

* \\(i, o\\) are the incoming and outgoing direction
* \\(h_r\\)  is the normalized half vector
* \\(n\\) is the macro-surface normal
* \\(F\\) is Fresnel term, \\(G\\) is masking/shadowing term and \\(D\\) is the microfacet normal distribution

> Please refer to the paper of Walter et al. [[2](#ref.2)] for more details.

## Evaluate the PDF for direction sampling

The job of `ggxEvalPdf` is to evaluate the PDF for given direction and **it must match the density** of samples generated by `ggxEvalSample`. The strategy suggested by Walter et al. [[2](#ref.2)] is to sample the microfacet normal \\(m\\) according to \\(p_m(m)=D(m)\vert m\cdot n \vert\\), and then transform the PDF from microfacet to macro-surface:

$$p_o(o)=p_m(m)\left\|\frac{\partial\omega_h}{\partial\omega_o}\right\|=\frac{D(m)|m⋅n|}{4|i⋅m|}$$

<pre class="prettyprint linenums lang-cpp">
float ggxEvalPdf(const void *brdfData, const AtVector *indir)
{
    // V = -sg->Rd, Nf = sg->Nf
    auto M = AiV3Normalize(V + *indir);
    auto IdotM = ABS(AiV3Dot(V, M));
    auto MdotN = ABS(AiV3Dot(M, Nf));
    // We pass Nf to D(), since it needs to compute thetaM which is the angle
    // between microfacet normal M and macro-surface normal N.
    return D(M, Nf) * MdotN * 0.25f / IdotM;
}
</pre>

## Create the sample direction from PDF
In `ggxEvalSample`, the direction is sampled with the cosine weighted microfacet normal distribution:

$$p_m (m)=D(m)|m⋅n|\text{, where }D(m)=\frac{\alpha_g^2}{\pi\cos^4{\theta_m}(\alpha_g^2+\tan^2{\theta_m})^2}$$

With the distribution transformation from \\(\omega_m\\) to \\(\theta_m, \phi_m\\):

$$p_m(\theta_m, \phi_m)=\sin{\theta_m}\ p_m(m)$$

We can compute the marginal probability of \\(p_m(\theta_m)\\) and conditional probability \\(p_m(\phi_m\vert\theta_m)\\)

$$p_m(\theta_m)=\int_0^{2\pi}{D(m)\cos{\theta_m} \sin{\theta_m}\ d\phi_m} = \frac{2\alpha_g^2\sin\theta_m}{\cos^3{\theta_m}(\alpha_g^2+\tan^2{\theta_m})^2}$$

$$p_m(\phi_m\vert\theta_m) = \frac{p_m(\theta_m, \phi_m)}{p_m(\theta_m)} = \frac{1}{2\pi}$$

Then, we compute the corresponding CDFs and their inverse functions to get \\(\theta_m, \phi_m\\) from two canonial random variable \\(\xi_1, \xi_2 \in [0, 1)\\)

<div>
$$\begin{aligned}
P_m(\theta_m) &= \int_0^{\theta_m} \frac{2 \alpha_g^2 \sin \theta'}{\cos^3{\theta'}(\alpha_g^2+\tan^2{\theta'})^2}\, d\theta' \\
&= -\frac{\alpha_g^2}{\alpha_g^2+\tan^2{\theta'}} \Big|_0^{\theta_m}\\
&= \frac{\tan^2{\theta_m}}{\alpha_g^2 + \tan^2{\theta_m}} \\
\theta_m &= \arctan \left( \alpha_g \sqrt{\frac{\xi_1}{1 - \xi_1}} \right)
\end{aligned}$$

$$P_m(\phi_m|\theta_m) = \int_0^{\phi_m} \frac{1}{2\pi} d\phi' = \frac{\phi_m}{2\pi}, \kern{1em} \phi_m = 2\pi \xi_2$$
</div>

Finally, we generate a sample direction with spherical coordinate \\(\theta_m, \phi_m\\) and transform the vector from local frame of shading point U, V, Nf(face-forward shading normal) into world space:

<pre class="prettyprint linenums lang-cpp">
AtVector ggxEvalSample(const void *brdfData, float rx, float ry)
{
    // Sample microfacet normal.
    auto thetaM = atanf(roughness * sqrtf(rx / (1.0f - rx)));
    auto phiM = AI_PITIMES2 * ry;

    AtVector M;
    M.z = cosf(thetaM);
    float r = sqrtf(1.0f - SQR(M.z));
    M.x = r * cosf(phiM);
    M.y = r * sinf(phiM);

    // Transform the M from local frame to world space.
    // The U, V can be retrieved from AiBuildLocalFramePolar with sg->Nf.
    AiV3RotateToFrame(M, U, V, Nf);

    // Compute reflected direction according to view direction and normal vector.
    // V = -sg->Rd
    return 2.0f * ABS(AiV3Dot(V, M)) * M - V;
}
</pre>

# Results
In comparison to aiStandard, GGX has stronger tails than Beckmann which is used in aiStandard.

<figure class="figure">
<a href="/images/rlShaders/rlGgx_roughness_cmp.png" data-lightbox="results"><img src="/images/rlShaders/rlGgx_roughness_cmp.png"></a>
<figcaption class="figure-caption">Roughness 0.0 to 1.0</figcaption>
</figure>

<a href="/images/rlShaders/rlGgx_buddha_cmp.jpg" data-lightbox="results">
![Buddha](/images/rlShaders/rlGgx_buddha_cmp.jpg)</a>

The results of the custom BRDF composed of GGX for glossy and Oren Nayar for diffuse.

<a href="/images/rlShaders/rlGgx_lucy.jpg" data-lightbox="results"><img src="/images/rlShaders/rlGgx_lucy.jpg"></a>

<a href="/images/rlShaders/rlGgx_lucy_in_grace_cathedral.jpg" data-lightbox="results"><img src="/images/rlShaders/rlGgx_lucy_in_grace_cathedral.jpg"></a>


# References

1. Veach, Eric, and Leonidas J. Guibas. <cite id="ref.1">"[Optimally combining sampling techniques for Monte Carlo rendering.](https://graphics.stanford.edu/papers/combine/)"</cite> SIGGRAPH, 1995.
2. Walter, Bruce, et al. <cite id="ref.2">"[Microfacet models for refraction through rough surfaces.](http://www.cs.cornell.edu/~srm/publications/EGSR07-btdf.html)"</cite> EGSR'07.
3. Colbert, Mark, Simon Premoze, and Guillaume François. "Importance sampling for production rendering." SIGGRAPH 2010 Course Notes.
4. Pharr, Matt, and Greg Humphreys. _Physically based rendering: From theory to implementation 2/e_. Morgan Kaufmann. 2010.
