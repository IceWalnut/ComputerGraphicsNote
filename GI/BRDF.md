# Unity Shader 入门精要 第六章 Unity 中的基础光照

以下摘自 "Unity Shader 入门精要" Ch6 "Unity 中的基础光照"



## 1.1 How do we watch the world

### 1.1.1 irradiance

- 平行光
    $$
    irradiance = E/(St)
    $$
    $l$ 是光的方向, $S$ 是垂直于 $l$ 的面积, $t$ 是时间, $E$ 是在 $t$ 时间内照射到 $S$ 面积上的能量.

    即, 对于平行光, $irradiance$ 可以通过计算在垂直于 $l$ 的单位面积上单位时间内穿过的能量来得到.

    如果光线方向和表面不垂直, 可以使用光源方向 $l$ 和 表面法线 $n$ 之间的夹角的余弦值得到. (默认矢量的模都为 1)

    ![image-20240116115035678](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202401161150743.png)

    右图中物体表面光线中的距离是 $d/cos\theta$

    因为辐照度是和照射到物体表面时光线之间的距离 $d/cos\theta$ 成反比的, 因此辐照度就和 $cos\theta$ 成正比. 

    $cos\theta$ 可以使用 <font color=red>光源方向 $l$ 和 表面法线 $n$ 的点积</font> 来得到. 这就是使用点积来计算辐照度的由来



### 1.1.2 吸收和散射

光线由光源发射出来后, 就会与一些物体相交. 通常, 相交的结果有两个: <font color=red>散射 scattering</font> 和 <font color=red>吸收 absorption</font>.

- scattering
    - Scattering will change the **direction** of the incident light, but doesn't change the **density** and **color** of incident light.
- absorption
    - Absorption will change the **density** and **color** of light rather than the **direction** of incident light.



After **scattering** on the surface of the object, the light will have two directions. The first will scatter inside of the object, which is called **refraction** or **transmission**. The second will scatter outside of the object, which is called **reflection**. For opaque objects, the light refracted into the interior of  the object continues to intersect with the particles inside. Some of the light will reflect outside the surface of the object, and some of it will be absorbed by the object.  The color and directions of the light which was emitted from the surface are different from those of the incident light.

![image-20240116151443341](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202401161514382.png)

In order to distinguish between those two different scattering directions, we use different parts in the **lighting model** to calculate them. <font color=red>**Specular**</font> part represents how object surface reflects light. <font color=red>**Diffuse**</font> part expresses how much the light is reflected, absorbed and scattered out of the surface.

According to the **direction and amount** of the incident light, we can calculate those of the **emergent light**, which are described as <font color=red>**exitance(出射度)**</font> usually. There is a **linear** relationship between <font color=blue>irradiance and exitance</font>, and their ratio is the diffuse reflection property and specular reflection property of the material.

In this chapter, we assume that the diffuse component is *non-directional*, which is to say, the light is evenly distributed in all directions. At the same time, we only consider the specular reflection in a specific direction.

> - refraction 折射
> - transmission 投射
> - lighting model 光照模型
> - specular 高光
> - diffuse 漫反射
> - exitance 出射度



### 1.1.3 Shading

<font color=red>**Shading**</font> refers to the process of  using a **equation** to calculate the **exitance** along a particular viewing direction based on material properties (such as diffuse reflection) and light source information (such as light direction and irradiance, etc). We also refer to this equation as the **Lighting Model**. Different lighting models serve different purposes. For instance, some are used to describe surfaces of rough objects, while others are used to describe metallic surfaces.

































































