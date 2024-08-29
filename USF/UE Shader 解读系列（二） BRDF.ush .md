# UE Shader 解读系列（二） BRDF.ush 

本文基于 UE 5.3.2 



## 1. 结构体 BxDFContext

```c++
struct BxDFContext
{
    half NoV;		// 法线向量和视角向量的点积，如果是归一化向量则是 cos \theta
    half NoL;		// 法线向量和光源向量夹角余弦值
    half VoL;		// 视角向量和光源向量夹角余弦值
    // 半角向量 H 是视角向量 V 与光源向量 L 的平均
    half NoH;		// 法线向量和半角向量 `H` 的点积
}
```



### 1.1 半角向量 H

通常被定义为视角向量 $V$ 和 光源向量 $L$ 的归一化和，即
$$
H = \frac{V + L}{|V + L|}
$$
这个向量代表观察方向和光源方向的 “中间” 方向。是一个归一化向量

- **镜面反射和半角方向的关系**

    对一个表面上的点，镜面反射最强的方向是当观察方向 $V$ 与反射方向 $R$ 完全重合的时候。如果 $V$ 与 $R$ 完全重合，那么镜面反射达到最大值。

    在很多光照模型（如 Blinn-Phong 模型）中，镜面反射的强度与法线 $N$ 和半角向量 $H$ 之间的夹角相关。具体来说，镜面反射项通常表示为
    $$
    \bold{Specular} = (N \cdot H)^n
    $$
    其中

    - $n$ 是**光滑度参数**
        - 控制反射的 **锐利程度**

    这个公式表示 <font color=red>法线和半角向量之间的夹角越小（即 $N$ 与 $H$ 越接近），镜面反射就越强</font>

    > 至于为什么，看下图或者这两个链接就清楚了 [入门Shading，详解Blinn-Phong和Phong光照模型 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/352209183)， [图形学基础-Phong和Blinn-Phong光照模型 | Molers Blog](https://molers.github.io/2021/06/30/图形学基础-Phong和Blinn-Phong光照模型/)
    >
    > <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408271614736.png" alt="image-20240827161410705" style="zoom:50%;" />



































