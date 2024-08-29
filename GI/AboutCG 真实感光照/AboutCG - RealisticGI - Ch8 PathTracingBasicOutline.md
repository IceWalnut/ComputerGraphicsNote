# AboutCG - RealisticGI Ch8 PathTracingBasicOutline



- tangent 空间

    - 切线空间是在做随机的时候经常用到的概念
    - 快速构建 tbn 的方法

- lambert 材质表面，lambert brdf

    - 这里假设整个表面都是漫反射表面，使用漫反射表面演示光路是如何传输的，后面的章节中再使用其他的复杂材质
    - 推导 brdf, pdf

- $\cos$ weight 随机采样

- 路径追踪渲染方程，路径追踪基础框架

    - 1次反弹

        - 光源经过一次反弹进入相机，先采样一条这样的路径

            <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407262028440.png" alt="image-20240726202827346" style="zoom:33%;" />

    - 多条路径

        - 使用 Monte Carlo 积分实现多条路径

            <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407262028861.png" alt="image-20240726202852779" style="zoom:33%;" />

    - 多次反弹

        - 最后经过多重反弹打到光源

            <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407262035056.png" alt="image-20240726203545993" style="zoom:33%;" />







## 1. Tangent Space 切线空间

- Tangent Space
    - 3 个 world 空间下的正交向量组成的空间
- 构建方法
    - fastTBN 舒密特正交
    - uv 



### 1.1 切线空间

关于切线空间 tangent space，[切线空间(Tangent Space) 的计算与应用](https://blog.csdn.net/u010385624/article/details/91994006) 这篇文章讲得非常好，我就不 copy 了。这里仅记录 AboutCG 讲师的讲解。

比如世界空间中如下图坐标，$(x,y)$ 坐标是不被束缚的。

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407271554636.png" alt="image-20240727155445518" style="zoom:50%;" />

路径追踪中，通常是在一个点的半球面内随机，随机的方向和法线是相关的，比如随机一个和法线夹角 <= 60 ° 的向量。如果换一个点，就需要如图再计算。在世界空间坐标中，计算会比较麻烦。

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407271641891.png" alt="image-20240727164154819" style="zoom:50%;" />

这里使用一个比较简单的方法，就是在局部空间中做随机，然后再局部空间转世界空间。这样只需要同一套算法做随机，如下图。

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407271649247.png" alt="image-20240727164949178" style="zoom:50%;" />

这个局部空间就叫做切线空间。

所谓的切线空间就是在世界坐标中找 3 个正交向量组成的空间



### 1.2 TBN 空间

如下图中和 法线 $\bold N$ 垂直的两个相互正交且和点相切的向量，连同法线一起组成了切线空间。红色的为主切线 $\bold T $ (**Tangent**)，蓝色的为副切线 $\bold B$ (**Bitangent**)。因此空间也叫做 <font color=red>**TBN 空间**</font>。

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407271947822.png" alt="image-20240727194744770" style="zoom:50%;" />

如此在 TBN 空间中取一个随机点 $(a, b,c)$ ，最后把这三个分量乘进三个轴中，就可以得到在世界空间中的坐标了



<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407271621642.svg" alt="Tangentialvektor" style="zoom:50%;" />

一个点的 TBN 空间不是唯一的。在球面上，每个点都有自己的 TBN 空间，它们互相独立。



### 1.3 fastTBN 舒密特正交

只要知道一个点的法线，就可以构建出一个切线空间，和旁边的点无关。现在用到的几何都是一些很简单的几何，如圆形、方块形。后面会用到一些更复杂的方法，如根据 **纹理坐标** 去求解切线空间。那时切线空间就不再互相独立，而是在一个三角形中，所有的点会共享一个 TBN。那时会根据纹理坐标和世界坐标计算出一个共享的切线空间，整个三角形会一起用这个空间。

现在还是来看 **舒密特正交法**

已知一个点的法线 $\bold N$，利用坐标互相正交的特点，随便找一个辅助向量 $\bold X$，（没错，就是随便一个向量都可以），然后利用叉积 
$$
\bold N \times \bold X = \bold T
$$
这样就得到了第一个主切线向量 $\bold T$

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407300908505.png" alt="image-20240730090810363" style="zoom:33%;" />

> 为什么？
>
> 因为 $\bold N$ 和 $\bold X$ 的叉积结果一定是垂直于 $\bold N$ 的，那么 $\bold T$ 一定在垂直于 $\bold N$ 的平面内，也就是该点的切平面内，那么 $\bold T$ 也就是 该点 的切线！

然后再用 
$$
\bold N \times \bold T = \bold B
$$
$\bold B$ 也垂直于 $\bold N$，也是切线，这样就得到一组正交基。



### 1.4 代码实现

- Coordinate.fx 

    Coordinate 是一个工具函数，这里加入切线空间的函数

    ```c++
    // @get_tangent_space
    // return 3x3 tbn
    // [tangent]
    // [normal]
    // [bitangent]
    float3x3 get_tangent_space(in const float3 n)
    {
    	// random vector X
      float3 UpVector = abs(n.z) < 0.999 ? float3(0, 0, 1) : float3(1, 0, 0);
      float3 TangentX = normalize(cross(UpVector, n));
      float3 TangentY = cross(n, TangentX);
      // 这里是 Y 轴朝上，所以是 TNB 空间
      return float3x3(TangentX, n, TangentY);
    }
    ```

    这个矩阵应该是这样子的
    $$
    \left [\begin{matrix}
    \color{red}T \\ \color{green}N \\ \color{blue}B  
    \end{matrix}
    \right]
    $$

接下来看怎么使用切线矩阵 TBN 矩阵



### 1.5 TBN 矩阵的使用

最常用的就是从切线空间内转换到世界空间

在 Coordinate.fx 中添加

```c++
// @tangent_to_world
// normalize vector from tangent to world
float3 tangent_to_world(in const float3 v, in const float3x3 tbn)
{
  // use [3x3]tbn mul [3x1]v
  return normalize(v.x * tbn[0] + v.y * tbn[1] + v.z * tbn[2]);
}
```

假设一个向量 $\bold v = [a, b, c]$，实际上这里就是得到 $\bold v$ 左乘一个 $[\bold T, \bold N, \bold B]^T$
$$
\left [\begin{matrix}
\color{red}T \\ \color{green}N \\ \color{blue}B  
\end{matrix}
\right] \
\left [\begin{matrix}
\bold v
\end{matrix}
\right]
$$
最后结果是 $a \bold T + b \bold N + c \bold B $，然后再归一化，注意这个结果是一个矢量而不是标量

因为 `tbn[i]` 是矢量不是标量

这就是从切线空间到世界空间的变换



## 2. BRDF 与 PDF 推导

- 漫反射表面 Lambert
    - BRDF
    - PDF



所谓的漫反射表面也就是 Lambert 表面，通俗地讲就是非常粗糙的表面，即在各个方向上任何入射都会均匀地反射入射光。这意味着从任何方向观测，该表面看起来都是一样亮的。

特点：半球面上任意一个方向，它的反射亮度都是相等的。那么漫反射的辐射亮度就和眼睛的位置无关。无论从哪个地方看这一点看到的颜色是一样的



### 2.1 BRDF 推导

假如一束光打到表面上的一点，然后沿着半球面上的任意一个方向射出。

![image-20240802182700519](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408021827627.png)

- $L_i$ 入射方向的辐射亮度
- $\mathrm dL_o$ 出射方向的辐射亮度微分
- $\theta$ 出射方向和法线的夹角

根据 BRDF 的定义，
$$
\mathrm dL_o = L_i \cdot BRDF \cdot \cos \theta \tag{8.2.1}
$$
因为漫反射的 BRDF 是一个常数，故
$$
\mathrm dL_o = L_i \cdot c \cdot \cos \theta \tag{8.2.2}
$$
为方便起见，我们直接令 $L_i = 1$，然后将上式两边积分，即将上图中半球面的每个方向积分 (更加具体的见 https://www.cnblogs.com/icewalnut/p/18321665 5.2 小节)
$$
\begin{aligned}
L_o &= \int_\Omega c \cdot \cos\theta \ \mathrm d \omega_i \\
&= c\int_0^{\pi/2} \cos\theta \sin\theta \ \mathrm d\theta \ \int_0^{2\pi} \ \mathrm d \phi \\
&= c\pi
\end{aligned}
$$
如果没有能量损失 $L_o = L_i$，则
$$
c = \frac{1}{\pi}
$$
然后后面还要再乘以 $color$，就是材质的颜色
$$
\frac{color}{\pi}
$$
就是完整的 Lambertian 漫反射 BRDF



接下来计算采样这个 Lambertian 漫反射的概率密度



### 2.2 PDF 推导

在第 7 章的反向路径追中里面，已知入射方向 $\omega_i$，想要知道下一个方向是哪一个，比较笨的方法是在半球面均匀随机方向，这样每个方向随机的概率是一样的

![image-20240803003319488](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408030033516.png)

但是这样是有问题的，因为从 $L_b$ 方向得到的辐射亮度的贡献是比其他方向大的

所有的辐射亮度进来的时候会乘以一个 $\cos \theta$。

法线方向的 $\cos\theta=1$，而和法线成 90° 的方向，$\cos\theta=0$。也就是说 $L_f$ 方向就没有必要分配路径。

要分配路径到 $\theta$ 比较小的地方。

如此就要重新分配每条路径的概率，也就是每个方向随机的概率是不一样的，大概如下图所示：









