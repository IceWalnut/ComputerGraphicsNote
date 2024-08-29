# Microfacet Models

Copied from [Microfacet Models (pbr-book.org)](https://www.pbr-book.org/3ed-2018/Reflection_Models/Microfacet_Models#fragbit-922)



Many geometric-optics-based approaches to modeling surface reflection and transmission are based on the idea that rough surfaces can be modeled as a collection of small **microfacets**. Surfaces comprised of microfacets are often modeled as **heightfileds**, where the distribution of facet orientations is described statistically. 

Figure 1 shows cross sections of a relatively rough surface and a much smoother microfacet surface. When the *distinction* isn't clear, we'll use the term **microsurface** to describe microfacet surfaces and **macrosurface** to describe the underlying smooth surface (e.g., as represented by a [`Shape`](https://www.pbr-book.org/3ed-2018/Shapes/Basic_Shape_Interface.html#Shape).)

> - distinction
>   - a difference between two similar things

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202404280806080.png" alt="image-20240428080614033" style="zoom: 50%;" />

<p style="font-size:15px"> Figure 1: Microfacet surface models are often described by a function that gives the distribution normals n_f with respect to the surface normal n. (a) The greater the variation of microfacet normals, the rougher the surface is. (b) Smooth surfaces have relatively little variation of microfacet normals.</p>



Microfacet-based BRDF models work by statistically modeling the scattering of light from a large collection of microfacets. If we assume that the differential area being illuminated, $\mathrm dA$, is relatively large compared to the size of individual microfacets, then a large number of microfacets are illuminated and it's their aggregate behavior that determines the observed scattering.

> - aggragate
>   - sth formed by adding together several amounts or things



- **Components of Microfacet Models**
  - A representation of the distribution of facets
  - A BRDF that describes how light scatters from individual microfacets

Given these, the task is to derive a closed-form expression giving the BRDF that describes scattering from such a surface. **Perfect mirror reflection** is most commonly used for the microfacet BRDF, though specular transmission is useful for modeling many translucent materials, and the **Oren–Nayar** model (described in the next section) treats microfacets as **Lambertian** reflectors.

To compute reflection from such a model, local lighting effects at the microfacet level need to be considered (Figure 2). Microfacets may be **occluded** by another facet, may lie in the **shadow** of a neighboring microfacet, or **interreflection** may cause a microfacet to reflect more light than predicted by the amount of direct illumination and the low-level microfacet BRDF. Particular microfacet-based BRDF models consider each of these effects with varying degrees of accuracy. The general approach is to make the best approximations possible, while still obtaining an easily evaluated expression.



![image-20240429161308103](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202404291613160.png)

<p style="font-size:15px">Figure 2: Three Important Geometric Effects to Consider with Microfacet Reflection Models. (a) Masking: the microfacet of interest isn't visible to the viewer due to occlusion by another microfacet. (b) Shadowing: analogously, light doesn't reach the microfacet. (c) Interreflection: light bounces among the microfacets before reaching the viewer.</p>



## 1.1 Oren-Nayar Diffuse Reflection

Oren and Nayar ([1994](https://www.pbr-book.org/3ed-2018/Reflection_Models/Further_Reading.html#cite:Oren94)) observed that real-world objects do not exhibit perfect Lambertian reflection. Specifically, rough surfaces generally appear brighter as the illumination direction approaches the viewing direction. They developed a reflection model that describes **rough surfaces by V-shaped microfacets** described by a spherical Gaussian distribution with a single parameter $\textcolor{red}\sigma$, **the standard deviation of the microfacet orientation angle**.

> - deviation
>   - the amount by which a measurement is different from a fixed number of amount 偏差
>   - standard deviation 标准差（方差的平方根）

Under the V-shape assumption, interreflection can be accounted for by only considering the neighboring microfacet; Oren and Nayar took advantage of this to derive a BRDF that models the aggregate reflection of the collection of grooves.

> - groove
>   - 凹槽

The resulting model, which accounts for shadowing, masking, and interreflection among the microfacets, does not have a closed-form solution, so they found the following approximation that fit it well:
$$
f_r(\omega_i, \omega_o) = \frac{R}{\pi}(A + B \max(0, \cos(\phi_i - \phi_o))\sin \alpha \tan\beta) \tag{1}
$$
where if $\sigma$ is in radians,

> - radian 弧度

$$
A = 1-\frac{\sigma^2}{2(\sigma^2 + 0.33)} \tag{2}
$$

$$
B = \frac{0.45 \sigma^2}{\sigma^2 + 0.09} \tag{3}
$$

$$
\alpha = max(\theta_i, \theta_o) \tag{4}
$$

$$
\beta = min(\theta_i, \theta_o) \tag{5}
$$

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202404300827642.png" alt="image-20240430082731475" style="zoom:50%;" /><img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202404300828936.png" alt="image-20240430082803781" style="zoom:50%;" />

<p style="font-size:15px">Figure 3: Dragon model rendered (a) with standard diffuse reflection from the Lambertian Reflection model and (b) with the OrenNayar model with a $\sigma$ parameter of 20°. Note the increase in reflection at the *silhouette* edges and the generally less-drawn-out transitions at light terminator edges with the Oren-Nayar model.</p>

> - silhouette
>   - 剪影
> - drawn-out
>   - 延长的，冗长的
>   - less-drawn-out 不那么明显
> - transition 过渡
>
> 注意使用 Oren-Nayar 模型时轮廓边缘的反射增加，以及光照终止边缘的过渡通常更少绘制。



The implementation here precomputes and stores the values of the $A$ and $B$ parameters in the constructor to save work in evaluating the BRDF later. Figure 3 compares the difference between rendering with **ideal diffuse reflection** and with the Oren-Nayar model.



### 1.1.1 OrenNayar Public Methods

```c++
OrenNayar(const Spectrum &R, Float sigma)
    : BxDF(BxDFType(BSDF_REFLECTION | BSDF_DIFFUSE)), R(R) 
	{
		sigma = Ridians(sigma);
    	Float sigma2 = sigma * sigma;
        A = 1.f - (sigma2 / (2.f * (sigma2 + 0.33f)));
        B = 0.45f * sigma2 / (sigma2 + 0.09f);
	}
```

[Spectrum](https://www.pbr-book.org/3ed-2018/Color_and_Radiometry/Spectral_Representation.html#Spectrum)

> - spectrum 光谱
>     - 这里是一个结构体，在上面的连接中有详细信息

[BxDF](https://www.pbr-book.org/3ed-2018/Reflection_Models/Basic_Interface.html#BxDF)

> - BRDFs and BTDFs share a common base class, [`BxDF`](https://www.pbr-book.org/3ed-2018/Reflection_Models/Basic_Interface#BxDF)



### 1.1.2 OrenNayar Private Data

```c++
const Spectrum R;
Float A, B;
```



Application of *trigonometric* identities can substantially improve the efficiency of the evaluation routine compared to a direct translation of underlying equations. The implementation starts by computing the values of $\sin\theta_i$ and $\sin\theta_o$.

> - trigonometric
>     - 三角学的；三角法的



### 1.1.3 BxDF Method Definitions

```c++
Spectrum OrenNayar::f(const Vector3f &wo, const Vector3f &wi) const
{
    Float sinThetaI = SinTheta(wi);
    Float sinThetaO = SinTheta(wo);
    ComputeCosineTermOfOrenNayarModel();
    ComputeSineAndTangentTermsOfOrenNayarModel();
    return R * InvPi * (A + B * maxCos * sinAlpha * tanBeta);
}
```

> - [`SinTheta`](https://www.pbr-book.org/3ed-2018/Reflection_Models.html#SinTheta)

To compute the $\max(0, \cos(\phi_i - \phi_o))$ term, we can apply the trigonometric identity
$$
\cos(a - b) = \cos a \cos b + \sin a \sin b \tag{6}
$$
such that we just need to compute the sines and cosines of $\phi_i$ and $\phi_o$.



### 1.1.4 Compute cosine term of Oren-Nayar model

```c++
Float maxCos = 0;
if (sinThetaI > 1e-4 && sinThetaO > 1e-4)
{
    Float sinPhiI = SinPhi(wi), cosPhiI = CosPhi(wi);
    Float sinPhiO = SinPhi(wo), cosPhiO = CosPhi(wo);
    Float dCos = cosPhiI * cosPhiO + sinPhiI * sinPhiO;
    maxCos = std::max((Float)0, dCos);
}
```

Finally, the $\sin\alpha$ and $\tan\beta$ terms are found. Note that whichever of $\omega_i$ or $\omega_o$ has a larger value for $\cos\theta$ (i.e., a larger $z$ component) has a smaller value for $\theta$. (这句话是说如果 $\theta$ 大则 $\cos\theta$ 小). We can set $\sin \alpha$ using the appropriate sine value computed at the beginning of the method. The tangent can then be computed using the identity $\tan\theta = \sin\theta / \cos\theta$.



### 1.1.5 Compute sine and tangent terms of Oren-Nayar Model

```c++
Float sinAlpha, tanBeta;
if (AbsCosTheta(wi) > AbsCosTheta(wo))
{
    sinAlpha = sinThetaO;
    tanBeta = sinThetaI / AbsCosTheta(wi);
}
else 
{
    sinAlpha = sinThetaI;
    tanBeta = sinThetaO / AbsCosTheta(wo);
}
```



## 2. Microfacet Distribution Functions

Reflection models based on microfacets that exhibit perfect specular relfection and transimission have been effective at modeling light scattering from a variety of glossy materials, including metals, plastic, and *frosted glass*. Before we discuss the radiometric details of these models, we'll first introduce abstractions that encapsulate their geometric properties. The code here includes implementations of two widely used microfacet models. All of this code is in the files [`core/microfacet.h`](https://github.com/mmp/pbrt-v3/tree/master/src/core/microfacet.h) and [`core/microfacet.cpp`](https://github.com/mmp/pbrt-v3/tree/master/src/core/microfacet.cpp).

> - frosted
>     - 霜冻的；无光泽的
>     - frosted glass 磨砂玻璃



**MicrofacetDistribution** defines the interface provided by microfacet implementations as well as some common functionality for them.



### MicrofacetDistribution Declarations

```c++
class MicrofacetDistribution
{
public:
    MicrofacetDistribution Public Methods
protected:
    MicrofacetDistribution Protected Methods
}
```



































































