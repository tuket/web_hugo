---
layout: post
title: BRDF based importance sampling for microfacets cheatsheet
date: 2017-07-09
use_math: true
---

<p align="center">
  <img src="http://i.imgur.com/HhN37cN.jpg" width="60%" height="60%" alt="[image]" >
</p>

BRDF based importance sampling aims to reduce the cost of computing the rendering equation by taking into account the BRDF itself.
In this case we are going to use the normal distribution function NDF[^sampling_rough].

First let's review the rendering equation (simplified version)

$$
  L_o(\vec{\omega} _ o)= L_e(\vec{\omega} _ o) + \int _\Omega {f_r(\vec{\omega} _i, \vec{\omega} _o) \ L_i(\vec{\omega} _i}) \ (\vec{\omega}_i \cdot \vec{n}) \ d\vec{\omega} _i
$$

$\vec{\omega}_i$: incoming directions.

$\vec{\omega}_o$: outgoing direction.

$L_i$: incoming energy per unit surface per [steradian](https://en.wikipedia.org/wiki/Steradian).

$L_o$: outgoing energy per steradian.

$f_r$: the BSDF or BRDF in our case. Is a statistical representation of the ratio outgoing energy / incoming energy.

$\vec{\omega}_i \cdot \vec{n}$: [view factor](https://en.wikipedia.org/wiki/Steradian).

$d\vec{\omega} _i$: infinitesimal area in the unit sphere along $\vec{\omega} _i$.
  

The BRDF we are going to focus on is based on microfacet theory[^cook]. If you want to learn more about microfacet theory check out [the course from Naty Hoffman](http://blog.selfshadow.com/publications/s2013-shading-course/hoffman/s2013_pbs_physics_math_notes.pdf).

## Cook-Torrance

The Cook-Torrance BRDF is based in microfacet theory:

$$
f_r(\vec\omega _i , \vec\omega _o, \vec n) =
  \frac{
     F(\vec \omega _o, \vec h)  \ G(\vec \omega _i, \vec \omega _o, \vec h) \ D(\vec h)
  }
  {
    4 \ \lvert \vec\omega _i \cdot \vec n \rvert \ \lvert \vec\omega _o \cdot \vec n \rvert
  }
$$

## $F$: Fresnel term
|                 |                 |
| :-------------: | :-------------: |
| Exact Fresnel   |  $\frac 1 2 \left(\frac{g-c}{g+c} \right) ^2 \\ \left(1 + \left( \frac{c \ (g + c) - 1} {c \ (g-c) +1 } \right)^2 \right)    \\\\ where: g = \sqrt{\frac{\eta _t} {\eta _i} ^2 -1 +c^2} \\ , \\ \\ \\ c = \lvert \vec\omega _o \cdot \vec h \rvert$  |
| Schlick's approximation      | $F_0 + (1-F_0) \\ (1 - cos \theta _h) ^ 5$      |
| Epic approximation | $ F_0 + (1-F_0) \\ 2 ^{(-5.55473 \\ (\vec\omega _o \cdot \vec h) - 6.98316) \\ (\vec\omega _o \cdot \vec h)} $ [^epic]      |

## $G$: geometry term (shadowing and masking)

Both $G$ and $D$ depend on the type of distribution.

$$
  G(\vec \omega _i, \vec \omega _o, \vec h) = G_1(\vec \omega _i, \vec h) \ G_1(\vec \omega _o, \vec h)
$$

| Distribution | $ G_1(\vec v, \vec h) $ |
| :---: | :---: |
| Beckmann | $$ \chi ^{+} \left(\frac {\vec v \cdot \vec h} {\vec v \cdot \vec n} \right)  \begin{dcases}  \frac {3.535a + 2.181a^2} {1 + 2.276a + 2.577a^2}   & if \ \ \ a < 1.6 \\\\ 1 & else \end{dcases} $$  <br/> with $ a= \frac 1 {\alpha _b \ \tan \theta _v} $ |
| Blinn-Phong | Same as Beckmann but with $ a= \frac {\sqrt{0.5 \alpha _p + 1}} {\tan \theta _v} $ |
| GGX | $$\chi ^{+} \left( \frac {\vec v \cdot \vec h} {\vec v \cdot \vec n} \right)  \frac 2 {1 + \sqrt{1 + (\alpha_g \\ \tan \theta_v)^2}}$$  |



$\alpha_b$: width parameter for the Beckmann distribution <br>
$\alpha_p$: Blinn-Phong exponent or shininess <br>
$\alpha_g$: width parameter for the GGX distribution, also know as squared roughness.

## $D$: normal distribution

If you are going to compute the rendering equation with **uniform sampling or sampling analytic lights** such as point lights, use one of these formulas:

| Distribution | $ D $ |
| :---: | :---: |
| Beckmann | $$ \frac{\chi^+ \left(\vec n \cdot \vec h \right)} {\pi \left( \alpha_b \cos^2 \theta_h \right)^2} e ^{- \left( \frac{\tan \theta_h} {\alpha_b} \right)}$$ |
| Blinn-Phong | $$ \chi^+ \left( \vec n \cdot \vec h \right) \frac {\alpha_p + 2} {2 \pi} \left( \cos  \theta_h \right) ^{\alpha_p}$$ |
| GGX | $$  \frac{\chi^+ \left( \vec n \cdot \vec h \right)} {\pi} \left( \frac {\alpha_g} {cos^2 \theta_h \left( \alpha _g ^2 + \tan^2 \theta_h \right)} \right)^2 = \frac{\chi^+ \left( \vec n \cdot \vec h \right)} {\pi} \left( \frac {\alpha_g} {cos^2 \theta_h \ \alpha _g ^2 + \sin^2 \theta_h } \right)^2 $$  |

With **importance sampling** we sample according to $D(\vec h) \cos \theta_h$.
Given two random numbers $ \xi_1 \xi_2 \ \in [0, 1)$ we compute $\vec h$ with one of these formulas:


| Distribution | $ \theta, \phi $ |
| :---: | :---: |
| Beckmann | $$ \theta_h = \arctan \left( \sqrt{-\alpha^2 _b \log(1 - \xi_1)} \right) \\ \phi_h = 2 \pi \xi_2 $$ |
| Blinn-Phong | $$ \theta_m = \arccos \left( \xi_1 ^{\frac{1}{\alpha_p+2}} \right) \\ \phi_m = 2 \pi \xi_2$$ |
| GGX | $$ \theta_m = \arctan \frac{\alpha_g \sqrt{\xi_1}} {\sqrt{1-\xi_1}} \\ \phi_m = 2 \pi \xi_2$$  |


Compute $\vec h$ from $\theta_h$ and $\phi_h$:

$$
\vec h = 
\begin{bmatrix}
\sin(\theta_h) \\ cos(\phi_h) \\\\ \sin(\theta_h) \\ sin(\phi_h) \\\\ \cos(\theta_h)
\end{bmatrix}$$

Then we have to compute the sampling direction from the outgoing direction and the half vector (in GLSL there is the **reflect** function):

$$ 
\vec \omega_i = 2 \lvert \vec h \cdot  \vec \omega_o \rvert \   \vec h - \vec \omega_o
$$

Since we are sampling with $f_r(\vec\omega _i , \vec\omega _o, \vec n) \ \lvert \vec\omega _i \cdot \vec n \rvert$ as the probability, then we divide the BRDF by that amount (more info in the references [^montecarlo] [^sampling_rough]). Leaving us with this new expression:

$$
w(\vec \omega_i) = \frac {f_r(\vec\omega _i , \vec\omega _o, \vec n) \ \lvert \vec\omega _i \cdot \vec n \rvert} {p_i(\vec\omega _i)}
= \frac{\lvert \vec \omega_o \cdot \vec h \rvert \ G(\vec \omega_i, \vec \omega_o, \vec h)} {\lvert \vec \omega_o \cdot \vec n \rvert \ \lvert \vec h \cdot \vec n \rvert}
$$

As you can see we have not only reduced variance but also the new expression is cheaper to compute. We would compute the rendering equation like this:

$$
  L_o(\vec{\omega} _ i)= L_e(\vec{\omega} _ i) + \int _\Omega {\ L_i(\vec{\omega} _i})
\frac{\lvert \vec \omega_o \cdot \vec h \rvert \ G(\vec \omega_i, \vec \omega_o, \vec h)} {\lvert \vec \omega_o \cdot \vec n \rvert \ \lvert \vec h \cdot \vec n \rvert}
 \ d\vec{\omega} _i
$$

## References

[^cook]: [A Reflectance Model for Computer Graphics](https://graphics.pixar.com/library/ReflectanceModel/paper.pdf)
[^sampling_rough]: [Microfacet Models for Refraction through Rough Surfaces](https://www.cs.cornell.edu/~srm/publications/EGSR07-btdf.pdf)
[^epic]: [Real Shading in Unreal Engine 4](http://blog.selfshadow.com/publications/s2013-shading-course/#course_content)
[^montecarlo]: [Monte Carlo theory: methods and examples - Chapter 9](http://statweb.stanford.edu/~owen/mc/Ch-var-is.pdf)
