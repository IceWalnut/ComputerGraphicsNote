# Global Illumination

Global illumination (GI) is a key concept in computer grahpics that refers to a set of techniques used to simulate how light **bounces off** surfaces and **scatters** throughout an environment, producing realistic lighting effects in rendered images or scenes.



The main idea behind global illunimation is to <font color=red>account not only for the light that comes directly from a light source (direct illumination), but also for the complex interactions that light has once it hits objects and surfaces</font>. These interactions include **reflection, refraction, and scattering**, which contribute to what we perceive as ***shadows, color bleeding (where colors from one object or surface subtly affect the surrounding objects), and the overall mood of a scene***.



Golbal illumination enhances the realism of 3D graphics by:

- **Softening shadows**

  - Unlike hard shadows that often produced by direct light alone, global illumination can create <font color=red>soft shadow edges</font>, which are more natural looking.

- **Adding <font color=red>indirect</font> light**

  - Areas not directly lit by a light source are still illuminated through light bouncing off other surfaces, which is particularly noticeable in corners and indoor scenes.

- **Enhancing colors**

  - GI allows colors to <font color=red>***bleed***</font> from one surface to another, adding richness and depth to the scene.

    > - bleed
    >   - here means diffuse 扩散



Implementing global illumination in rendering can be challenging due to its computational intensity. Various algorithms and techniques have been developed  to simulate GI, including:

- **Ray tracing**
    - Simulates the path of light rays as they bounce around the scene, but can be <font color=blue>computationally expensive</font>.

- **Radiosity**

    - Focuses on how light is distributed across surfaces, particularly good for static scenes.

        > - radiosity
        >     - 光能传递，热辐射

- **Photon mapping**

    - Stores information about how photons (representing light) interact with surfaces, balancing between efficiency and accuracy

        > - photon
        >     - = light quantum 光子

- **Screen space global illumination (SSGI)**
    - A more <font color=blue>performance-friendly</font> approach used in realtime applications like video games, approximating GI effects without extensive computation.

Each method has its advantage and trade-offs, making them suitable for different types of projects, from real-time video games to high-quality movie rendering where realism is paramount.









# Video on Bilibili

The following contents are copied from the video [Games 104 Modern Game Engine 21. Dynamic GI and Lumen](https://www.bilibili.com/video/BV1oe411u7DJ).

![image-20240321172343113](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403211723216.png)

The most complexed thing is **Integration**. How to solve the problem?



## 1. 基础知识

### 1.1 Monte Carlo Integration

![image-20240321172420587](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403211724618.png)

- Approximate integral with the average of randomly sample values

  ![image-20240321215142688](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403212151728.png)

做 Rendering 就是在半球上进行积分，进行采样，对于 Shading 来讲就是把所有方向的光，也就是 Shading 的每个 radiance 全部算出来然后加在一起



### 1.2 Monte Carlo Ray Tracing (Offline)

![image-20240321215505837](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403212155944.png)

这个方法原理非常简单，就是光线不停地反射，这样每次 bounce，之后计算的复杂度是指数级增长的。所以说做 single bounce 的话还可以，但是做 multi bounce 就非常困难。

这里面还有个有趣的点，这张图没有讲清楚，就是实际上场景里面，真正亮的地方，是非常非常 **不均匀** 的。像康奈尔这种整体照亮的 case 实际上是很特殊的。在很多场景里面，比如说一个硕大无比的房间，它真正亮的地方，可能就是天顶的窗子那一点。但是整个房间就被这个 multi bounce 的光照亮了。这时你在进行 sampling ，你会发现，比如，第一次 bounce ，射出 50 根 ray，第二次 50 * 50 = 2500， 第三次更多，但是，只有极少数的 ray 打中天顶上的窗户，所以它根本没采到光。。。所以 Monte Carlo 在过去的几十年中，它的一生之敌就是 **采样**。如果采样不好，因为是随机的，所以没办法保证 pixel a 和 pixel b 形成的采样的结果是连续的，就会产生大量的 **noise**。

过去用 GPU，一帧渲染几个小时，就是这个原因，因为你要做 Monte Carlo RayTracing。所以最痛苦的事情就是 sampling



### 1.3 Sampling is the Key

![image-20240321220608457](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403212206603.png)

Lumen 要解决的核心问题，就是怎么更好地 sampling，怎么有效地分布这些 sampling 的问题。

基于 Monte Carlo ，sampling 的数量是越多越好，但是 system overhead 就越大，它就会越慢。

上图中左上角是 1 根 ray，从上到下，从左到右都是 double 的，所以右下角已经是 $2^{15} = 32768$ 根 ray 的时候，已经是很 smooth 了

> 这里王希讲的是 $2^{16}=65532$ 根 ray，但是我觉得如果第一张图是 $2^0$ 的话右下角是 $2^{15}$

但是如果是 single bounce，几分钟可能渲染出来，如果是 multi bounce，可能需要很久才能渲染出来。

<font color=red>如何 sampling 是 Monte Carlo Base 的 GI 的核心点</font>。



## 2. Sampling 怎么做

### 2.1 Sampling: Uniform Sampling 均匀采样

![image-20240322070318650](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403220703679.png)
$$
\begin{aligned}
\displaystyle \int^{b}_{a}f(x) \ dx &= \displaystyle \lim_{n \rarr \infty} \frac {1}{n} \displaystyle \sum^{n}_{i=1}f(x_i)(b-a) \\
&= \displaystyle \lim_{n \rarr \infty} \frac{1}{n} \displaystyle \sum ^{n}_{i=1} \frac{f(x_i)}{\textcolor{red}{\frac{1}{b-a}}}
\end{aligned}
$$
其中 $\frac {1}{b-a}$ 被称为 <font color=red>**Probability Dentity Function**</font> 概率密度函数

- We are doing uniform random sample, so we have $\frac {1}{b-a}$ factor here.

但是，如果信号是不均匀的，除非 sampling 的密度很高，否则对采样率不能很高效地利用。比如说我想 sample 空间上的一个光场，对光照进行积分，在 uniform sample 情况下，如果两根 ray 之间有夹角，ray 射出 10 米之外，采样线之间的夹角很可能会把天顶的窗户整个漏掉了。这就是 Uniform Sampling 的一个问题。



### 2.2 Probability Distribution Function

概率分布函数，还是叫 `Probability Density Function` 概率密度函数？

![image-20240322164307321](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403221643375.png)
$$
\displaystyle \int f(x) \ dx \ \sim \ F_n(X)=\frac{1}{n} \displaystyle \sum ^n_{k=1}\frac{f(X_k)}{\textcolor{red}{PDF(X_k)}}
$$
Probability Distribution Function

- Describes the relative likehood for this random variable to take on a given value. 描述该随机变量取给定值的相对可能性

    > - likehood function 似然函数

- Higher means more possible to be chosen

如果采样也是按照这个函数来分布的话，我就可能用尽可能少的采样来尽可能逼近于它想要的真实水平



### 2.3 Importance Sampling

The PDF can be arbitrary, but which is the best?

![image-20240322171012535](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403221710566.png)

更接近真实积分。比如说，计算 Shading 的时候，进可能朝着光比较亮的地方，沿着法线正对的地方，去多射一些采样的 ray，这样可以用少量的 ray 来获取想要的结果。

在过去，所有做 GI 的方法，比如 [Radiosity 算法](https://en.wikipedia.org/wiki/Radiosity_(computer_graphics))，Monte Carlo RayTracing，Photon Mapping 等算法提出来之后，都会有衍生研究提出来，怎么去做它的 importance sampling。也就是在同样的计算时间下，或者同样的采样率下，能达到一个更好的，更稳定的，更 smooth 的效果。



### 2.4 Importance Sampling: Best PDF for Rendering?

- Rendering equation:

$$
L_o(p,\omega_o)=\displaystyle \int _{\Omega^+} L_i(p, \omega_i)f_r(p, \omega_i, \omega_o)(n \cdot \omega_i) \mathrm{d}\omega_i
$$

- Monte Carlo Integration:

$$
L_o(p, \omega_o) \approx \frac{1}{N} \displaystyle \sum_{i=1}^{N} \frac{L_i(p, \omega_i)f_r(p, \omega_i, \omega_o)(n \cdot \omega_i)}{p(\omega_i)}
$$

- What's our $f(x)$
  $$
  L_i(p, \omega_i)f_r(p,\omega_i,\omega_o)(n \cdot \omega_i)
  $$

- <font color=red>What's our pdf </font>
  
  - Uniform: $\textcolor{red}{p(\omega_i)= \displaystyle \frac{1}{2\pi}}$
  - Other pdf ? (cosine-weight, GGX)



> 关于公式
>
> $$
>L_o(p, \omega_o) \approx \frac {1}{N} \displaystyle \sum_{i=1}^{N} \frac{L_i(p, \omega_i)f_r(p, \omega_i, \omega_o)(n \cdot \omega_i)}{p(\omega_i)}
> $$
> 
> 也就是蒙特卡洛近似的渲染方程，可以看 《渲染方程.md》

做 rendering 的时候 pdf 怎么选呢？如果大部分的反射可以假设为 diffuse 面的话，那它对光的敏感度基本满足于一个 diffuse loop，就是靠近于 normal 的方向就敏感一点，就是 cosine。如果是很侧面射来的光，即使光线很强，感应度也一般。



### 2.5 Importance Sampling: PDF is Matter

如果在天空中均匀地采样，进行光场的积分，可以发现，同样是 256 sampling per pixel (spp)，uniform sampling 有很多噪点。但是如果按照 cosine loop 来分布，即靠近天顶的地方密一点，靠近下面稀疏一点。此时采样的噪点就会下降很多

![image-20240324181809445](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403241818503.png)

- $p(\omega)=\displaystyle \frac{1}{2\pi}$

![image-20240324182103287](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403241821314.png)

- $\textcolor{red}{p(\omega)=\displaystyle \frac{\cos \theta}{\pi}}$

  ![image-20240324182234752](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403241822782.png)





### 2.6 Importance Sampling : Cosine and GGX PDF

对于非常 glossy 的东西，像符合 GGX 材质特点的。

> - GGX 材质特点
>     - 像音响一样，高频足够尖，足够高，低频足够宽。

- Cosine PDF

$$
p(\omega)= \frac{\cos \theta}{\pi}
$$

![image-20240327184856240](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403271848265.png)

![image-20240327184647888](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202403271846946.png)



- GGX PDF

$$
p(\omega)= \frac{\alpha^2 \cos \theta}{\pi((\alpha^2-1) \cos^2{\theta}+1)^2}
$$

简单来讲 GGX 的 loop 会比 cosine 更加 sharp 一点。如果基于 normal，基于 GGX 进行优化的话，实际上能够更好地抓住 glossy 的效果。但是这是一个比较 specific 的东西，不具有普遍意义。

这一 part 主要是让大家理解， GI 的核心是，要找到一个好的 sampling 的方法，用尽可能少的 ray 获取四面八方的光场对你的影响的一个表达。

我对每个屏幕上的一个小的像素点，这是一个 mesh，我需要知道，上下左右的光是怎么样的？我就发出 ray 去问，这个问，就讲究要问在正确的地方。

这里讲的 sampling 只讲了表面材质自己，其实还跟光有关系。



## 3. Reflective Shadow Maps (RSM, 2005)

Let's inject light in. (Photon Mapping?)

前面讲的这些 Monte Carlo 算法基本上都是离线算法，在电影或者其他行业运用。在游戏行业的话 overhead 太大了。

RSM 是一系列算法的开山鼻祖，启发了很多人。让大家敢于想象，realtime 的 GI 也是可以做的。

解决的核心问题是：怎么把光注入到场景中去。

GI 有两类方法，一类是 Monte Carlo Raytracing，就是不停地 bounce

还有一种方法：<font color=red>**Photon Mapping**</font>。以前都是从光源射出来的光子，打到了物体上，不停地反弹，人眼收集到了反弹的光子。所以 ray tracing 都是眼睛倒着去找。现在可以正着去找，从光的角度看。光源 cast 了无数的 photon，这些 photon 不停地 bounce，到了物体表面，某些就会停留在物体表面，最后 shading 的时候，就是停留在物体表面的 photon 的 radiance，也就是物体表面那一点的光的分布。Shading 的时候，就是在 gather 好的 photon 上面进行收集，插值，最后给出这个 shading。这就是 photon mapping 的核心思想。



### 3.1 什么是 Reflective Shadow Map

基于 observation，我们在渲染阴影的时候会渲一帧 shadowmap。Shadowmap 和正常的渲染最大的不同是什么？

正常的渲染，是从相机视角来看；shadowmap，实际上是从光的视角来看的。所以说在渲 shadowmap 的时候只取了一个 depth，如果在 shadowmap 的 光 的位置的话，要把场景整个渲一遍，如果要把它沿着光 shading 一遍的话，如果没有 GI 只有 direct lighting 的话，则所有被照亮的面都在那个 map 里面。

> 此处王希觉得这个术语命名不好，叫 illumination map 或者 radiance map 都行

就是从光的位置看到的所有被照亮的表面（没有 multi bounce），那张图里看到的，如果是一个 spot light，也就是锥形，所有被第一次照亮的表面。

RSM 就是基于这个想法。

如果现在知道，在空间中所有被照亮的点以及它被第一次照亮时的亮度，这个光它可以 scatter 出来。那么在渲染任何一个人眼睛看到的点的时候，会把空间中被照亮的点散射出来的 radiance 全部收集一下，就可以照亮别的点了。













- Each pixel on the shadow map is a indirect light source





























