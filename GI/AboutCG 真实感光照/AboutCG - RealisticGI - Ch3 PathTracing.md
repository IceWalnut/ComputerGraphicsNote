# AboutCG RealisticGI - Ch3 路径追踪



- 路径追踪的原理

- 改写渲染方程
  - 基于路径的积分形式
  - 立体角和面积的转换



![image-20240611235955152](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406112359210.png)





[The Light Transport Equation (pbr-book.org)](https://www.pbr-book.org/3ed-2018/Light_Transport_I_Surface_Reflection/The_Light_Transport_Equation)



## 1. 介绍

### 1. 光线追踪

![image-20240711083100695](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407110831723.png)

如图，从眼睛出发，求 x 点的辐射亮度，于是这个点就开始沿着半球面的每个方向找，找到每个方向的入射亮度，然后用 brdf
$$
L_o(p, \omega_o) = \int_\Omega f(p, \omega_o, \omega_i)L_i(p, \omega_i) |\cos\theta_i| \ d\omega_i \tag{3.1.1} 
$$
但是这个渲染方程是有问题的，入射的辐射亮度 $L_i(p, \omega_i)$ 可以继续展开，也能展开一个渲染方程。这样就出现了递归，有了先有鸡还是先有蛋的问题。要实现这个方程就需要阉割掉一些特性。

如此，提出了一种更好的方法，即 **路径追踪**。

路径追踪会改写渲染方程

现在式 (3.1.1) 中是以立体角积分的形式，后面会改成基于路径积分的形式。改写的时候会用到立体角和面积相互转换的公式。





## 2. 路径照度



![image-20240711223906345](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407112239452.png)

窗户中的光射到地板上，地板打到人的脸上，然后再到摄像机

这就是路径追踪



## 3. 路径微分

第二章中就是路径追踪的第一灵感。要计算矩形光源对人面部的影响，需要微分，于是先计算一个点，即 dA。

地板上和人脸上都是由许多点组成的。把这 3 点连起来，最后把人脸上的点连接到相机，相机 + 3 个点组成了一个长度为 3 的路径。



![image-20240711224304014](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407112243125.png)

上图中显示了长度为 3 的路径。注意是一些点。

路径追踪就是 **在场景中寻找一些点，把这些点连接起来形成路径，就只需要算这些点形成的路径对相机的辐射亮度的贡献。找到一条之后就能找到很多其他的路径。问题就是需要找到全部这些路径。**

现在问题来了，相机看到下面这一点，怎么找到和光源连接的所有的路径？

![image-20240711224618862](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407112246978.png)

拆分这个问题，路径是有长度的，长度可以是2， 3， 4， 5，。。。我们可以找到长度为 2,3,4,5,6,7,8,9... 的路径

比如，先寻找长度为 1 的路径，也就是场景中随机一个点，如下图中的三个点，长度都为 1

![image-20240711224831460](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407112248567.png)

但是最左边的点，并没有贡献辐射亮度，它的贡献为 0，另外两个点，假设它们的贡献都是 1，这 3 条路径加起来，总贡献为 2。

再来看长度为 2 的所有路径，如下图中举例两条，左边一条，右边一条

![image-20240711225031809](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407112250917.png)

长度为 3 或者 4 的类似，就不一一举例了。

所以路径追踪的算法，就是

先找出所有长度为 1 的路径的点，也就是场景中随机 1 个点；再找到场景中随机两个点，三个点。。。

思路非常简答， 相较于光线追踪，它的优点在于：

1. **寻找的路径永远是单向的，不用担心循环递归问题**
2. 光线追踪中，随机的是 **半球面的每一个方向**；而路径追踪中，随机的是场景中的一个点





## 4. 路径追踪渲染方程

[PBR-book - The Light Transport Equation](https://www.pbr-book.org/3ed-2018/Light_Transport_I_Surface_Reflection/The_Light_Transport_Equation)

### 4.1 原文摘抄

The **light transport equation (LTE)** is the governing equation that describes the equilibrium distribution of radiance in a scene. It gives the total reflected at a point on a surface in terms of emission from the surface, its **BSDF**, and the distribution of **incident illumination** arriving at the point. For now we will continue only to consider the case where there are no participating media (参与介质) in the scene. (Chapter 15 describes the generalizations to this process necessary for scenes that do have participating media.)



The detial that makes evaluating the LTE difficult is the fact that incident radiance at a point is affected by the <font color=purple>**geometry and scattering properties**</font> of all of the objects in the scene. For example, a bright light shining on a red object may cause a reddish tint on nearby objects in the scene, or glass may focus light into caustic (蚀刻的) patterns on a tabletop. Rendering algorithms that account for this complexity are often called **global illumination algorithms**, to differentiate them from **local illumination ** algorithms that use only information about the local surface properties in their shading computations.



In this section, we will first derive the LTE and describe some approaches for manipulating the equation to make it easier to solve numerically. We will then describe two generalizations of the LTE that make some of its key properties more clear and serve as the foundation for some of the advanced integrators that will be implemented in Chapter 8.



#### 4.1.1 Basic Derivation

The light transport equation depends on the basic assumptions we have already made in choosing to use radiometry to describe light--that wave optics effects are unimportant and that the distribution of radiance in the scene is in equilibrium.

> - assumption
>     - 假设
> - equilibrium 
>     - 平衡

The key principle underlying the LTE is ***energy blance***. Any change in energy has to be "charged" to some process, and we must keep track of all the energy. Since we are assuming that lighting is a linear process, the difference between the amount of energy going out and energy coming in of a system must alos be equal to the difference between energy emitted and energy absorbed. This idea holds at many levels of scale. On a macro level we have conservation of power:
$$
\Phi_o - \Phi_i = \Phi_e - \Phi_a \tag{4.1.1}
$$

> - e 
>     - emits
> - a
>     - absorb

In order to enforce energy balance at a surface, exitant radiance $L_o$ must be equal to emitted radiance plus the fraction of incident radiance that is scattered. Emitted radiance is given by $L_e$, and scattered radiance is given by the scattering equation, which gives
$$
L_o(p, \omega_o) = L_e(p, \omega_o) + \int_{S^2}f(p, \omega_o, \omega_i)L(p, \omega_i)|\cos\theta_i| \ \mathrm d\omega_i \tag{4.1.2}
$$
Because we have assumed for now that no participating media are present, radiance is constant along rays through the  scene. We can therefore ralate the incident radiance at $p$ to the outgoing radiance from another $p'$, as show by Figure 4.1. If we define the **ray-casting function <font color=red>$t(p, \omega)$</font>** as a function that computes the first surface point $p'$ intersected by a ray from $p$ in the direction $\omega$, we can write the incident radiance at $p$ in terms of outgoing radiance at $p'$:
$$
L_i(p, \omega) = L_o(t(p, \omega), -\omega) \tag{4.1.3}
$$
> 重点：$p'= t(p, \omega)$, $\omega_i = -\omega_o$

In case the scene is not closed, we will define the ray-casting function to return a special value $\Lambda$ if the ray $(p, \omega)$ doesn't intersect any object in the scene, such that $L_o(\Lambda, \omega)$ is always 0.

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407241018302.png" alt="image-20240724101812215" style="zoom:50%;" />

<p style="text-align:center;color:dimgrey;font-size:smaller">Figure 4.1 : Radiance along a Ray through Free Space Is Unchanged. Therefore, to copute the incident radiance along a ray from point p in direction \omega, we can find the first surface the ray intersects and compute exitant radiance in the direction - \omega there. The ray-casting function t(p, \omega) gives the point p' on the first surface that the ray (p, \omega) intersects.</p>



Dropping the subscripts from $L_o$ for brevity, this relationship allows us to write the LTE as
$$
L(p, \omega_o) = L_e(p, \omega_o) + \int_{S^2}f(p, \omega_o, \omega_i)L(t(p, \omega_i), -\omega_i) |\cos\theta_i| \mathrm d\omega_i \tag{4.1.4}
$$

> - brevity
>     - 简洁



The key to the above representation is that there is only one quantity of interest, exitant radiance from points on surfaces. Of cource, it appears on both sides of the equation, so our task is still not simple, but it is certainly better. It is important to keep in mind that we were able to arrive at this equation simply by enforcing energy balance in our scene.



#### 4.1.2 Analytic Solutions to the LTE

It is possible to find analytic solutions to the LTE in extremely simple settings. While this is of little help for general-purpose rendering, it can help with debugging the implementations of integrators. If an integrator that is supposed to solve the complete LTE doesn't compute a solution that matches an analytic solution, then clearly there is a bug in the integrator. As an example, consider the interior of a sphere where all points on the surface of the sphere have a <font color=red>**Lambertian BRDF**</font>, 
$$
f(p, \omega_o, \omega_i) = c \tag{4.1.5}
$$
> 兰伯特 BRDF 假设表面是理想的漫反射表面，即在各个方向上任何入射均匀地反射入射光。这意味着，从任何方向观测，该表面看起来都是一样亮的。
>
> - **兰伯特反射定律**
>
>     - 理想的漫反射表面（即兰伯特表面）会使光均匀地散射到所有方向。表面的亮度（或者说从表面反射的光强度）是恒定的，不随观察角度的变化而变化。具体来说，表面的反射光强度 $I$ 是与入射光的余弦值成正比
>         $$
>         I = I_i \cos\theta_i
>         $$
>
>         - $I_i$ 是入射光的强度
>         - $\theta_i$ 是入射光线和表面法线之间的角度
>
>     - Lambertian 定律的一个重要结果是：
>
>         - 尽管每单位面积的入射光通量与入射角度的余弦成正比，但由于表面在不同角度的可视面积也按余弦变化，所以观察者看到的亮度是均匀的
>
> - **兰伯特 BRDF**
>
>     - Lambertian BRDF 是 Lambertian 反射定律的一个定量化的表达，用来描述表面在不同方向上反射光的分布特性。<font color=red>兰伯特 BRDF 的数学表达式为</font>：
>         $$
>         \color{red}{f(p,\omega_o, \omega_i)= \displaystyle \frac{\rho}{\pi}}
>         $$
>
>         - $f(p, \omega_o, \omega_i)$ 是 BRDF，描述入射方向 $\omega_i$ 和 出射方向 $\omega_o$ 下的反射比例
>
>         - **$\rho$ 是表面的反射率（反照率）**，**表示表面反射的能量比例**
>
>         - $\pi$ 是由于能量守恒而引入的归一化因子。为什么是 $\pi$ ？ 简单来说是半球面积分 
>             $$
>             \displaystyle \int_{2\pi} \cos\theta \ \mathrm d\omega = \int_0^{\pi/2} \cos\theta\sin\theta \ \mathrm d\theta \int_0^{2\pi}\mathrm d\phi=\pi
>             $$
>             而反射的总能量不应该超过入射的总能量，因此，对于漫反射表面，我们期望满足：
>             $$
>             \int_\Omega f(p, \omega_o, \omega_i) \cos\theta_o \mathrm d\omega_o = 1
>             $$
>             (为什么是这个式子？因为上式再乘以 $L_e$ 就是 $L_o$ 了)
>
>             假设 $f(p, \omega_o, \omega_i) = \rho/x$ 为常数，其中 $\rho$ 为发射率，表示反射的总能量比例，那么
>             $$
>             \int_\Omega \frac{\rho}{x} \cos\theta_o d\omega_o = \frac{\rho}{x} \cdot \pi= 1 \quad \Rarr \quad x=\pi
>             $$
>             

and also emit a constant amount radiance in all directions. We have 
$$
L(p, \omega_o) = L_e + c \displaystyle \int_{H^2(n)} L(t(p, \omega_i), -\omega_i) |\cos \theta_i| \mathrm d\omega_i \tag{4.1.6}
$$


The outgoing radiance distribution at any point on the sphere interior must be the same as at any other point; nothing in the environment could introduce any variation among different points. **Therefore, the incident radiance distribution must be the same at all points, and the consine-weighted integral of incident radiance must be the same everywhere as well.** As such, we can replace the radiance functions with constants and simplify, writing the LTE as 
$$
L = L_e + c\pi L \tag{4.1.7}
$$
While we could immediately solve this equation for $L$, it's interesting to consider successive substitution of the right-hand side into the $L$ term on the right-hand side. If we also <font color=red>replace $\pi c$ with $\rho_{hh}$ </font>, the reflectance of a **Lambertian surface**, we have 
$$
\begin {aligned}
L &= L_e + \rho_{hh}(L_e + \rho_{hh} (L_e + ...)) \\
&= L_e + \rho_{hh}L_e + \rho^2_{hh}L_e + ... \\
&= \displaystyle \sum_{i = 0}^\infty L_e \rho_{hh}^i \cdot
\end{aligned} \tag{4.1.8}
$$
In other words, exitant radiance is equal to the emitted radiance at the point plus light that has been scattered by a BSDF once after emission, plus light that has been scattered twice, and so forth.

> - so forth
>     - so on, and so on



Because $\rho_{hh} < 1$ due to conservation of energy, the series converges and the reflected radiance at all points in all directions is
$$
L = \displaystyle \frac{L_e}{1-\rho_{hh}} \tag{4.1.9}
$$
> - converge
>     - 收敛；汇聚

This process of repeatedly substituting the LTE's right-hand side into incident radiance term in the integral can be instructive in more general cases. For example, the **DirectLightingIntegrator** integrator effectively computes the result of making a single substitution:
$$
L(p, \omega_o) = L_e(p, \omega_o) + \int_{S^2} f(p, \omega_o, \omega_i) L_d |\cos\theta_i| \mathrm d\omega_i \tag{4.1.10}
$$
where 
$$
L_d = L_e(t(p, \omega_i), -\omega_i) \tag{4.1.11}
$$
and further scattering is ignored.



Over the next few pages, we will see how performing successive substitutions in this manner and then regrouping the results expresses the LTE in a more natural way for developing rendering algorithms.



#### 4.1.3 The Surface Form of the LTE

One reason why the LTE as written in Equation (4.1.2)
$$
L(p, \omega_o) = L_e(p, \omega_o) + \int_{S^2}f(p, \omega_o, \omega_i) L(t(p, \omega_i), -\omega_i) |\cos \theta_i| \mathrm d\omega_i
$$
is complex is that the ralationship between geometric objects in the scene is implicit in the **ray-tracing function** $t(p, \omega)$. Making the behavior of this function explicit in the integrand will shed some light on the structure of this equation. To do this, we will rewrite Equation (4.1.2) as an integral over **area** instead of an integral over **directions** on the sphere.

> - shed
>     -  遮挡；去除

First, we define exitant radiance from a point $p'$ to a point $p$ by
$$
L(p' \rarr p) = L(p', \omega) \tag{4.1.12}
$$
if $p'$ and $p$ are mutually visible and $\omega = \widehat{p - p'}$. We can also write the BSDF at $p'$ as 
$$
f(p'' \rarr p' \rarr p) = f(p', \omega_o, \omega_i) \tag{4.1.13}
$$
where $\omega_i = \widehat{p'' - p}$ and $\omega_o = \widehat{p' - p}$ (Figure 4.2)



<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407221507750.png" alt="image-20240722150739657" style="zoom:50%;" />

<p style="text-align:center;color:dimgrey;font-size:smaller">Figure 4.2 The three-point form of the light transport equation converts the integral to be over the domain of points on surfaces in the scene, rather than over directions over the sphere. It is a key transformation for deriving the path integral form of the light transport equation.</p>



Rewriting the terms in the LTE in this manner isn't quite enough, however. We also need to multiply by the **Jacobian** that relates solid angle to area in order to transform the LTE from an integral over direction to one over surface area. Recall that this is $\displaystyle \frac{|\cos \theta'|}{r^2}$.

> 然而，以这种方式重写 LTE 中的项还是不够。我们还需要乘以将 **立体角** 和 **面积** 联系起来的雅克比行列式，以便将 LTE 从方向上的积分转换为表面积上的积分，就是  $\displaystyle \frac{|\cos\theta'|}{r^2}$
>
> 因为 
> $$
> \color {red}{\mathrm d\omega = \displaystyle \frac{\mathrm dA \ \cos \theta}{r^2} }\tag{4.1.14}
> $$

We will combine this change-of-variables term, the original $|\cos \theta|$ term of the LTE, and also a binary visibility function $V$ ($V = 1$ if the two points are mutually visible (两个点相互可见), and $V=0$ otherwise)
$$
V = \left\{
\begin{aligned}
& 1, && The \ two \ points \ are \ mutually \ visible\\
& 0, && Otherwise
\end{aligned}
\right. 
$$
into a single geometric compling term, $G(p \lrarr p')$:
$$
\color{red} {G(p \lrarr p') = V(p \lrarr p') \frac{|\cos\theta||\cos\theta'|}{||p - p'||^2}} \tag{4.1.15}
$$
Substituting these into the light transport equation and converting to an area integral, we have 
$$
L(p' \rarr p) = L_e(p' \rarr p) + \int _A f(p'' \rarr p' \rarr p)L(p'' \rarr p')G(p'' \lrarr p') \ \mathrm dA(p'') \tag{4.1.16}
$$
<font color=red>where $A$ is all of the surfaces of the scene</font>.

Althgough Equations (4.1.2) 
$$
L(p, \omega_o) = L_e(p, \omega_o) + \int_{S^2}f(p, \omega_o, \omega_i)L(t(p, \omega_i), -\omega_i) |\cos\theta| \ \mathrm d\omega_i
$$
and (4.1.16) are equivalent, they represent two different ways of approaching light transport. To evaluate Equation (4.1.2) with Monte Carlo, we would sample a number of <font color=red>**directions**</font> from a distribution of directions on the sphere and cast rays to evaluate the integrand. For Equation (4.1.16), however, we would choose a number of <font color=red>**point**</font> on surfaces according to a distribution over surface area and compute the coupling between those points to evaluate the intefrand, tracing rays to evaluate the visibility term $V(p \lrarr p')$.



#### 4.1.4 Integral over Paths

With the area integral form of Equation (4.1.16), we can derive a more flexible form of the LTE known as the **path integral** formulation of light transport, which expresses radiance as an integral over paths that are themselves points in a high dimensional **path space**. One of the main motivations for using path space is that it provides an expression for the value of a measurement as an explicit integral over paths, as opposed to the unwieldly recursive definition resulting from the energy balance equation, (4.1.2).

The explicit form allows for considerable freedom in how these paths are found--essentially any technique for randomly choosing paths can be turned into a workable rendering algorithm that computes the right answer given a sufficient number of samples. This form of the LTE provides the foundation of the bidirectional light transport algorithms in Chapter 16.

To go from the area integral to a sum over path integrals involving light-carrying paths of different lengths, we can now start to expand the three-point light transport equation, repeatedly substituting the right-hand side of the equation into the $L(p'' \rarr p')$ term inside the integral. Here are the first few terms that give incident radiance at a point $p_0$ from another point $p_1$, where $p_1$ is the first point on a surface along the ray from $p_0$ in direction $p_1 - p_0$:
$$
\begin {aligned}
L(p_1 \rarr p_0) &= L_e(p_1 \rarr p_0)\\
& \ \ \ \ + \int_A L_e(p_2 \rarr p_1) f(p_2 \rarr p_1 \rarr p_0) G(p_2 \rarr p_1) \mathrm dA(p_2) \\
& \ \ \ \ + \int_A \int_A L_e(p_3 \rarr p_2) f(p_3 \rarr p_2 \rarr p_1) G(p_3 \lrarr p_2)\\
\ & \ \ \ \ \ \ \ \  \times f(p_2 \rarr p_1 \rarr p_0) G(p_2 \lrarr p_1) \mathrm dA(p_3) \mathrm dA(p_2) + ... 
\end {aligned} \tag{4.1.17}
$$

> 公式 (4.1.17) 是由 (4.1.8) 
> $$
> L = \displaystyle \sum_{i=1}^\infty L_e \rho_{hh}^i
> $$
> 来的，其中，$L_e$ 是在每个点都不同的，而 $\rho_{hh}$ 原本是代替 $\pi c$ 的，$c$ 是 兰伯特 BRDF 也就是 $f(p, \omega_o, \omega_i)$，而 $\pi$ 是因为使用了 $\cos$ 积分得到的常数代替，因此，这里的 $\rho$ 其实就是上面的
> $$
> \int_A f(p_2 \rarr p_1 \rarr p_0)G(p_2 \rarr p_1) \mathrm dA(p_2)
> $$
> 以及
> $$
> \int_A f(p_3 \rarr p_2 \rarr p_1)G(p_3 \rarr p_2) \mathrm dA(p_3)
> $$
> 当 $\rho_{hh}$ 和 $\rho_{hh}$ 相乘时，就变为
> $$
> \int_A\int_A f(p_3 \rarr p_2 \rarr p_1)G(p_3 \lrarr p2) \times f(p_2 \rarr p_1 \rarr p_0) G(p_2 \lrarr p_1) \ \mathrm dA(p_2) \mathrm dA(p_3)
> $$
> $\mathrm dA(p_2)$ 和 $\mathrm dA(p_3)$ 是面积微分

Each time on the right side of this equation represents a path of increasing length. For example, the third term is illustrated in Figure 4.3. This path has four **vertices**, connected by three segments. The total contribution of all such paths of length four (i.e., a vertex at the camera, two vertices at points on surfaces in the scene, and a vertex on a light source) is given by this term. Here, the first two vertices of the path, $p_0$ and $p_1$, are predetermined based on the camera ray origin and the point that the camera ray intersects, but $p_2$ and $p_3$ can vary over all points on surfaces in the scene. The integral over all such $p_2$ and $p_3$ gives the total contribution of paths of length four to radiance arriving at the camera.

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407230835300.png" alt="image-20240723083552184" style="zoom:50%;" />

<p style="text-align:center; color: dimgrey; font-size: smaller">Figure 4.3 The integral over all points p_2 and p_3 on surfaces in the scene given by the light transport equation gives the total contribution of two bounce paths to radiance leaving p_1 in the direction of p_0. The components of the product in the integrand are shown here: the emitted radiance from the light, L_e; the geometric terms between vertices, G; and scattering from the BSDFs, f.</p>

This infinite sum can be written compactly as 
$$
L(p_1 \rarr p_0) = \sum_{n = 1}^{\infty}P(\bar p_n) \tag{4.1.18}
$$
<font color=red>$P(\bar p_n)$ gives the amount of radiance scattered over path $\bar p_n$ with $n + 1$ vertices</font>,
$$
\bar p_n = p_0, p_1,..., p_n
$$
where 

- $p_0$ 
    - film plane or front lens element 
- $p_n$
    - light source

$$
\begin{aligned}
P(\bar p_n) &= \int_A\int_A ... \int_A L_e(p_n \rarr p_{n-1}) \\
& \quad \quad \times \bigg(\displaystyle \prod_{i=1}^{n-1} f(p_{i+1} \rarr p_i \rarr p_{i-1})G(p_{i+1} \lrarr p_i) \bigg) \mathrm dA(p_2) ... \mathrm dA(p_n)
\end{aligned} \tag{4.1.19}
$$

Before we move on, we will define one additional term that will be helpful in the subsequent(随后的) discussion. The product of a path's BSDF and geometry terms is called the **throughput** of the path; it describes the fraction of radiance from the light source that arrives at the camera after all of the scattering at vertices between them. We will denote(标志) it by
$$
T(\bar p_n) = \displaystyle \prod_{i=1}^{n-1}f(p_{i+1} \rarr p_i \rarr p_{i-1})G(p_{i + 1} \lrarr p_i) \tag{4.1.20}
$$

> 可以认为 $T_n$ 就是整条路径的吞吐量

so
$$
P(\bar p_n) = \int_A\int_A...\int_A L_e(p_n \rarr p_{n-1})T(\bar p_n) \mathrm dA(p_2) ... \mathrm dA(p_n) \tag{4.1.21}
$$
Given equation (4.1.17) and a particular length $n$, all that we need to do to compute a Monte Carlo estimate of the radiance arriving at $p_0$ due to paths of length $n$ is to sample a set of vertices with an appropriate sampling density in the scene to generate a path and then to evaluate an estimate of $P(\bar p_n)$ using those vertices. Whether we generate those vertices by starting a path from the camera, starting from the light, starting from both ends, or starting from a point in the middle is a detail that only effects how the <font color=red>**weights** for the Monte Carlo estimates </font> are computed. We will see how this formulation leads to practical light transport algorithms throughout this and the following two chapters.



### 4.2 中文解释

下面是中文对上面的大致解释

已有的渲染方程为 
$$
L_o(p, \omega_o) = \int_{S^2} f(p, \omega_o, \omega_i) L_i(p, \omega_i) |\cos \theta_i| d\omega_i \tag{4.2.1}
$$
接下来看路径追踪

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407231044556.png" alt="image-20240723104444472" style="zoom:33%;" />

抽象为

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407221507750.png" alt="image-20240722150739657" style="zoom:50%;" />

p 点为眼睛，现在看长度为 2 的情况。p'' 点所在的平面为光源。所有的光源点都在 此平面内，连接这些点和 p' 点形成一个向量 $\omega_i$ ，将这个入射向量代入渲染方程
$$
L_o(p, \omega_o) = \int_{S^2} f(p, \omega_o, \omega_i) L_i(p, \omega_i) |\cos \theta_i| \ \mathrm d\omega_i
$$
可以看到 BRDF $f(p, \omega_o, \omega_i)$ 中的 p 没变, 射出方向 $\omega_o$ 没变, 摄入方向 $\omega_i$ 没变，入射的辐射亮度 $L_i(p, \omega_i)$ 也不用改，入射角 $\theta_i$ 也不用改，需要改的只有  $\mathrm d \omega_i$ 和 $\displaystyle \int_{S^2}$

也就是说，现在 <font color=red>**积分的主角从半球面中的立体角变成了光源面积上的一些点**</font>

所以 **半球面上的立体角微分也要改成面积上的点微分**

在 [pbr-book ch5.5](https://www.pbr-book.org/3ed-2018/Color_and_Radiometry/Working_with_Radiometric_Integrals) 中有介绍

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407231214950.png" alt="image-20240723121443842" style="zoom: 50%;" />

<p style="text-align:center;color:dimgrey;font-size:smaller">Figure 4.4 To compute irradiance at a point p from a quadrilateral source, it's easier to integrate over the surface area of the source than to integrate over the irregular set of directions that it subtends. The relationship between solid angles and areas given by Equation (d\omega = \frac{dA \cos\theta}{r^2}) let us go back and forth between the two approaches.</p>

渲染方程中 $\int_{S^2}$ 改为  $\int_A$， $\mathrm d\omega_i$ 改为 $\displaystyle \frac{dA\cos\theta}{r^2}$ ，就有了
$$
L_o(p, \omega_o) = \int_A f(p, \omega_o, \omega_i)L_i(p, \omega_i) |\cos \theta_i| \frac{dA\cos \theta}{r^2} \tag{4.2.2}
$$
然后用 $V(p'' \lrarr p')$ 来表示可见性，可见为 1，不可见为 0，有
$$
L_o(p, \omega_o) = \int_A f(p, \omega_o, \omega_i)L_i(p, \omega_i)|\cos\theta_i|V(p'' \lrarr p') \frac{dA\cos\theta}{r^2} \tag{4.2.3}
$$
 然后把 $|\cos\theta_i|V(p'' \lrarr p') \displaystyle \frac{dA\cos\theta}{r^2}$ **抽离出来为 <font color=red>几何 geometry</font>**
$$
G(p \lrarr p') = V(p \lrarr p') \frac{|\cos\theta||\cos \theta'|}{||p - p'||^2} \tag{4.2.4}
$$
令
$$
f(p'' \rarr p' \rarr p) = f(p', \omega_o, \omega_i) \tag{4.2.5}
$$

$$
L(p' \rarr p) = L(p', \omega) \tag{4.2.6}
$$

公式 (4.2.4)(4.2.5)(4.2.6) 代入 LTE 公式  (4.1.2) 得
$$
L(p' \rarr p) = L_e(p' \rarr p) + \int _A f(p'' \rarr p' \rarr p)L(p'' \rarr p')G(p'' \lrarr p) \mathrm dA(p'') \tag{4.2.7}
$$
也就是 公式 (4.1.16)



## 5. 路径追踪

[pbr-book 14.5 Path Tracing](https://www.pbr-book.org/3ed-2018/Light_Transport_I_Surface_Reflection/Path_Tracing)

在 AboutCG Ch3.5 的前半段就是对 pbr-book 14.4 中的公式推导

老师给出的中文总结：

### 5.1 路径追踪的核心思想

- <font color=red>**基于面积的辐射亮度投影**</font>
    - i.e. 可以假想为从光源出发，光源可以微分为一些点，这些点 ($\mathrm dA$)经过多次投影后形成路径，最后打到相机。
    - 路径是单向的不会死循环，解决了光线追踪的无穷递归的问题
- <font color=red>**以路径的辐射亮度为贡献**</font>
    - 计算整条路径的光线传输 (radiance)
- <font color=red>**路径积分的渲染方程**</font>
    - 知道一条路径的辐射亮度贡献，渲染方程也就能改写为基于路径的渲染方程
    - 它是基于面积为分的积分
    - 采样路径足够多，就能够近似结果

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407241711303.png" alt="image-20240724171151166" style="zoom:50%;" />



**方程**：

路径长度为 1 的方程
$$
L(p' \rarr p) = L_e(p' \rarr p) + \int_A f(p'' \rarr p' \rarr p) L(p'' \rarr p') G(p'' \lrarr p') \ \mathrm dA(p'')
$$

$$
L(p_1 \rarr p_0) = \displaystyle \sum_{n=1}^\infty P(\bar p_n)
$$

$P(\bar p_n)$ gives the amount of radiance scattered over a path $\bar p_n$ with $n + 1$ vertices
$$
P(\bar p_n) = \int_A \int_A ... \int_A L_e(p_n \rarr p_{n-1}) T(\bar p_n) \mathrm dA(p_2)...\mathrm dA(p_n)
$$

$$
T(\bar p_n) = \displaystyle \prod_{i=1}^{n-1}f(p_{i+1} \rarr p_i \rarr p_{i-1}) G(p_{i + 1} \lrarr p_i)
$$





### 5.2 路径追踪的算法

- 把场景中所有的点 $\mathrm dA$ 看做样本，随机选择 n 个点连接出路径
- 计算这些路径的 radiance 贡献
- 样本空间是场景中的点，累积所有路径的贡献



### 5.3 如何随机路径

所有长度 = 1 的路径

所有长度 = 2 的路径

所有长度 = n 的路径

put together ...



总结
$$
P(\bar p_n) = \int _A \int _A ...\int_A L_e(p_n \rarr p_{n-1})T(\bar p_n) \mathrm dA(p_2)...\mathrm dA(p_n) \\
T(\bar p_n) = \displaystyle \prod_{i=1}^n f(p_{i+1} \rarr p_i \rarr p_{i-1})G(p_{i+1} \lrarr p_i) \\
G(p_{i+1} \lrarr p_{i}) = V(p_{i + 1} \lrarr p_i) \frac{|\cos\theta||\cos \theta'|}{||p_{i + 1} - p_i||^2}
$$








































