# UE Shader 解读系列（一） ShadingCommon.ush 

本文基于 UE 5.3.2 

是对 ChatGPT 给的解释进行的 copy ，不一定正确

`ShadingCommon.ush` 包含了 UE 的 shader 编写相关的基础定义和工具函数



## 1. 宏定义与条件编译

```c++
#ifndef SHADING_PATH_MOBILE
#define SHADING_PATH_MOBILE 0
#endif

#ifndef SHADING_PATH_DEFERRED
#define SHADING_PATH_DEFERRED 0
#endif
```

这些宏定义用于确定渲染路径（如移动设备上的前向渲染 `SHADING_PATH_MOBILE` 和 延迟渲染 `SHADING_PATH_DEFERRED`）。`#ifndef` 用于检查这些宏是否已经定义，如果没有定义，则为它们分配默认值 0



## 2. Shading Models 着色模型

```c++
// SHADINGMODELID_* occupy the 4 low bits of an 8bit channel and SKIP_* occupy the 4 high bits
#define SHADINGMODELID_UNLIT				0
#define SHADINGMODELID_DEFALUT_LIT			1
#define SHADINGMODELID_SUBSURFACE			2
#define SHADINGMODELID_PREINTEGRATED_SKIN	3
#define SHADINGMODELID_CLEAR_COAT			4
#define SHADINGMODELID_SUBSURFACE_PROFILE	5
#define SHADINGMODELID_TWOSIDED_FOLIAGE		6
#define SHADINGMODELID_HAIR					7
#define SHADINGMODELID_CLOTH				8
#define SHADINGMODELID_EYE					9
#define SHADINGMODELID_SINGLELAYERWATER		10
#define	SHADINGMODELID_THIN_TRANSLUCENT		11
#define	SHADINGMODELID_STRATA				12

// TODO: fix begin
#define SHADINGMODELID_QTOON				13
// TODO: fix end

#define SHADINGMODELID_NUM					14
#define	SHADINGMODELID_MASK					0xF	// 4 bits
```

`SHADINGMODELID_*` 宏代表的 shadingmodel ID 占用了 8 位通道中的低 4 位，而 `SKIP_*` 宏占用了高 4 位。

在计算机图形学中， 8 位通道可以表示 0 到 255 的值，通常用于存储颜色、材质属性等数据。在这种情况下，一个 8 位的通道被分成两部分来存储不同的信息：

1. **低 4 位 (bit 0 - 3)**

    用于存储 shading model 的 ID 

    由于只有 4 位，因此可以表示的范围是 0 - 15，对应最多 16 个不同的 shading model

2. **高 4 位 (bit 4 - 7)**

    用于存储 `SKIP_*` 标志。这些标志可能用于控制某些渲染特性或优化行为。如跳过阴影、速度通道等渲染操作。



#### 位操作示例

假设一个 8 位数值是 `1101 0110`（十六进制表示为 `D6`）：

- **低 4 位 (`0110`)** 对应 `SHADINGMODELID_*`，值为 `6`。
- **高 4 位 (`1101`)** 对应 `SKIP_*` 标志。

通过使用位操作，可以从一个 8 位通道中提取出这两部分信息，用于不同的渲染处理逻辑。这种做法可以高效地利用存储空间和计算资源。



### 2.1 掩码 MASK

最后一个定义 `#define SHADINGMODELID_MASK 0xF` 是十进制的 15，十六进制的 `0xF`，二进制的 `1111`

每一位十六进制数正好对应 4 位二进制数，如 

- `0xF` 在二进制中是 `1111`，占用 4 位
- `0xFF` 在二进制中是 `1111 1111`，占用 8 位

通过使用 `0xF` ，直观看到它占用了低 4 位，正好对应着 `SHADINGMODELID` 用的 4 位（高四位是 `SKIP_*` 用的）



- **掩码 MASK**
    - <font color=red>**用于在位操作中筛选或屏蔽特定的位**</font>。掩码通常用于以下操作：
        - **提取**
            - 通过与 mask 进行 **与 `&`** 操作，可以提取出某些特定的值
        - **设置**
            - 通过 **或 `|`** 操作，可以设置特定位的值
        - **清除**
            - 通过与 **反掩码 `~MASK`** 进行 **与 `&`** 操作，可以清除特定位的值



### 2.2 `SHADINGMODELID_MASK` 的作用

`SHADINGMODELID_MASK` 被定义为 `0xF` ，即 `00001111`，表示二进制中的 低 4 位，这意味着在使用这个掩码时，只会影响这 4 位，例如

```c++
uint_8t shadingModelAndFlags = 0x9F; // 1001 1111
uint_8t shadingModelID = shadingModelAndFlags & SHADINGMODELID_MASK;
```

在这个例子中：

- `shadingModelAndFlags` 是一个包含着色模型 ID 和一些其他标志的 8 位值
- `shadingModelID` 是通过 `SHADINGMODELID_MASK` 提取出的 低 4 位值，结果是 `0xF` ，也就是 `1111` （十进制 15）



通过使用 `SHADINGMODELID_MASK` ，你可以确保只提取低 4 位的 `ShadingModelID` ，而不会受其它高位信息的干扰。



### 2.3 高位掩码

继续 `ShadingCommon.ush` 中的代码

```c++
// The flags are defined so that 0 value has no effect!
// These occupy the 4 high bits in the same channel as the SHADINGMODELID_*
#define HAS_ANISOTROPY_MASK			(1 << 4)
#define SKIP_PRECSHADOW_MASK		(1 << 5)
#define ZERO_PRECSHADOW_MASK		(1 << 6)
#define SKIP_VELOCITY_MASK			(1 << 7)

#define SSS_PROFILE_ID_INVALID  256
#define SSS_PROFILE_ID_PERPIXEL 512
```

- `HAS_ANISOTROPY_MASK`
  - `00010000` 判断是否启用了各向异性 Anisotropy 渲染
- `SKIP_PRECSHADOW_MASK`
  - `00100000` 跳过前置阴影计算
- `ZERO_PRECSHADOW_MASK`
  - `01000000` 强制前置阴影为零
- `SKIP_VELOCITY_MASK`
  - `10000000` 跳过速度计算（通常与运动模糊相关）

> 前置阴影（Precomputed Shadows，或称Precise Shadows）通常指的是在渲染过程中提前计算或预处理的阴影信息。相较于在每帧实时计算阴影，前置阴影的计算方式可以在渲染前的一个阶段进行，保存下来供后续渲染时使用，从而提高渲染效率，特别是在处理复杂的阴影效果时。
>
> ### 具体解释：
>
> 1. **预计算的阴影**：
>    - 在一些场景中，特别是静态场景或光照变化不大的情况下，阴影可以在渲染前提前计算。这些预计算的阴影信息可以存储在纹理或缓冲区中，供实时渲染时直接使用。
>    - 预计算阴影通常适用于固定光源和静态物体的场景。
> 2. **实时阴影与前置阴影的区别**：
>    - 实时阴影：每一帧都根据当前光源位置、物体位置和相机视角动态计算阴影，这通常非常耗费计算资源。
>    - 前置阴影：阴影信息提前计算并存储，因此在渲染时可以避免重复计算，只需在渲染时查表或读取预处理的阴影数据，提升性能。
> 3. **前置阴影在Unreal Engine中的应用**：
>    - 在某些复杂的着色模型或特定的渲染管线中，可以通过前置阴影技术来减少实时计算的开销。前置阴影可以被用来处理光照和阴影的细节，从而优化渲染性能。
>    - 例如，在定义位掩码时，可以选择是否跳过前置阴影的计算（如`SKIP_PRECSHADOW_MASK`），这意味着在某些情况下，渲染过程会忽略前置阴影，或者使用预先计算好的阴影数据，而不是每帧重新计算阴影。
>
> ### 总结：
>
> 前置阴影是一种用于优化渲染的技术，通过预先计算和存储阴影数据，可以在实时渲染中减少计算量，从而提升整体性能。这在处理复杂场景或要求高性能的应用中非常有用。



### 2.4 SSS 配置文件 ID 常量

```c++
#define SSS_PROFILE_ID_INVALID	256
#deinfe SSS_PROFILE_ID_PERPIXEL 512
```

1. `SSS_PROFILE_ID_INVALID`

   表示无效的次表面散射 （SSS）配置文件 ID，数值为 256

   当某个像素不需要使用 SSS 效果时，可以使用这个 ID

2. `SSS_PROFILE_ID_PERPIXEL`

   表示按像素的次表面散射配置文件 ID，数值为 512

   用于细粒度控制 SSS 效果的渲染



## 3. 获取 ShadingModel 颜色

```c++
// for dibugging and to visualize
float3 GetShadingModelColor(uint ShadingModelID)
{
    // TODO: PS4 doesn't optimize out correctly the switch(), so it thinks it needs all the Samples even if they get compiled out
#if PS4_PROFILE
    ...
#else
    switch(ShadingModelID)
    {
        case SHADINGMODELID_UNLIT: return float3(0.1f, 0.1f, 0.2f);	// Dark Blue
        case SHADINGMODELID_LIT: return float3(0.1f, 1.0f, 0.1f); // Green
            ...
                default: return float3(1.0f, 1.0f, 1.0f); // White
    }
#endif
}
```

这个函数根据传入的 `ShadingModelID` 返回相应的颜色（用于调试或可视化）。在 `PS4_PROFILE` 下，这些 ShadingModel 使用 `if-else` 来判断，在其它平台用 `switch-case` 语句。不同的 ShadingModelID 对应不同的颜色值。



## 4. Shading 模型的反面光照

```c++
#define SHADINGMODEL_REQUIRES_BACKFACE_LIGHTING (MATERIAL_SHADINGMODEL_TWOSIDED_FOLIAGE || STRATA_ENABLED)

bool GetShadingModelRequiresBackfaceLighting(uint ShadingModelID)
{
    return ShadingModelID == SHADINGMODELID_TWOSIDED_FOLIAGE;
}
```

这里定义了一个宏 `SHADINGMODEL_REQUIRES_BACKFACE_LIGHTING`

用于判断材质是否需要反面光照。`GetShadingModelRequiresBackfaceLighting` 函数判断是否是 **双面叶子材质**，如果是则需要反面光照。



## 5. F0 和 IOR （折射率）的转换

这些函数负责材质中 `F0` （材质的反射系数）和 `IOR` （材质的折射率）之间的转换。

### 5.1 F0ToDielectricSpecular

```c++
// Shading parameterisation
float F0ToDielectricSpecular(float F0)
{
    return saturate(F0 / 0.08f);
}
```

1. **F0 与反射率的关系**

    **`F0` 代表法线方向（即入射角为 0°）上的反射率**，表示光线在与表面法线垂直的角度（入射角为0°）入射时，被表面反射的光的比例。这个值与材料的折射率 (Index of Refraction, IOR) 有关。

    通过 菲涅尔方程 Fresnel Equation 算出来的

2. **DielectricSpecular**

    **电介质高光系数**。为什么用这个，因为通常金属是自然有高光的，电介质（即非金属，如玻璃、塑料等）的高光反射特性与金属反射特性不同。金属的高光反射特性受颜色影响，而 Dielectric Material 的高光反射通常是无色的（即白色或灰色）

3. **Saturate** 函数

    > - saturate
    >     - 浸透；充斥；饱和

    `saturate(x)` 是将值限制在 `[0, 1]` 范围内

    - if (x < 0) return 0; 
    - if (x > 1) return 1;
    - if (0 <= x <= 1) return x;

    在这个函数中，`saturate(F0 / 0.08f)` 确保最终结果在  `[0, 1]` 的范围内

4. **为什么用 `F0 / 0.08f`**

    <font color=red>0.08f 约等于折射率为 1.5 的电介质材料的 F0 值，这个值对应常见材料（如玻璃或者塑料）的典型 F0 反射率</font>。通过除以 `0.08f` ，这个函数将 `F0` 转换为一个规格化的高光反射系数 `DielectricSpecular` ，可以在渲染中直接使用。
    
    > 这里是 ChatGPT 给出的解释，这里复习一下，
    > $$
    > F_0 = \bigg( \frac{n -1}{n+1} \bigg)^2
    > $$
    > 如果 $n=1.5$，则 $F_0 = 0.04$ 不是 0.08
    >
    > - **ChatGPT 的解释**
    >
    > 使用 Fresnel 方程 计算得出的反射率，对于折射率 n=1.5 的材料，确实得出的 F0 值是 0.04，而不是 0.08。
    >
    > - **为什么用 0.08f**
    >
    > 0.08f 是渲染引擎中的一个经验值，它并不是直接来源于 Fresnel 方程的理论计算，而是一个在实际应用中确定的常数。这个值的使用是为了简化计算并适应常见的渲染需求，尤其是在基于物理的渲染（PBR）模型中。
    >
    > - **背后的原因**
    >
    > 1. **可用范围**: 在 PBR 中，许多电介质材料（例如塑料、木材、皮肤）通常具有非常接近的 F0F_0F0 值，这个值在 0.04 到 0.08 之间。使用 0.08f 作为基准值，可以更好地涵盖和调整这些材料的反射率范围。
    > 2. **增强视觉效果**: 使用 0.08f 可以使得渲染效果更加明亮和锐利，特别是在应用 specular 高光时。这种调整是基于视觉效果优化的，而不仅仅是物理精度。
    > 3. **经验调整**: 在实际开发中，经过大量的实验和调优，发现 0.08f 更适合实际的渲染需求，尤其是在处理各种材质的 specular 时，可以提供更一致的视觉表现。
    >
    > - **总结**
    >
    > 虽然理论上对于折射率 n=1.5 的材料，F0 应该是 0.04，但是在渲染引擎中选择使用 0.08f 作为经验值，是为了更好地适应视觉效果和材质范围的调整。在实际应用中，物理精确性和视觉效果之间常常需要权衡，而 0.08f 是这种权衡的结果。



### 5.2 F0RGBToF0

```c++
float F0RGBToF0(float3 F0)
{
    return dot(0.3333333.xxx, F0);
}
```

1. **函数功能**

    - 将一个 RGB 颜色表示的反射率 `F0` （`float3` 类型）转换为一个单一的标量 `F0` （`float`类型）。
    - 通过对 RGB 三个分量进行加权平均来计算单一的反射率值。

2. **参数**

    - `float3 F0`

        - 一个三维向量，表示 RGB 颜色空间中的反射率值。通常 `F0.r` 表示红色通道的反射率，`F0.g` 表示绿色通道的反射率，`F0.b` 表示蓝色通道的反射率

        >在计算机图形学和光学中，RGB 颜色表示的反射率通常指的是一个表面的颜色与光线交互后反射出的光的颜色。具体来说，每个 RGB 颜色通道（红色、绿色、蓝色）都表示光谱中相应波长的光的反射率。这种反射率描述了表面对不同颜色光的反射程度。
        >
        >**实际意义：**
        >
        >1. **材质的外观**：RGB 颜色表示了材质在光线照射下的外观。例如，一个高反射率的红色通道（R值高）意味着该材质在红光下反射较多，呈现出红色的外观。
        >2. **光线追踪与渲染**：在光线追踪或其他渲染技术中，RGB 反射率用于计算光线与材质表面的交互，决定最终图像中物体的颜色和亮度。
        >3. **能量守恒**：在物理上，反射率通常是一个 0 到 1 之间的值（或者 0% 到 100%），表示反射的光量与入射光量的比例。对于 RGB 颜色，如果反射率值超过 1（或 RGB 颜色值超过 255），表示了一个非物理的、不可能存在的材料，因为没有材料能够反射比入射更多的光。
        >4. **颜色校正和调节**：在实际应用中，理解 RGB 颜色的反射率可以帮助调整颜色，使其在不同的光照条件下表现一致或达到期望的效果。
        >
        >总结来说，RGB 颜色的反射率表示的是一个表面对不同颜色光的反射能力，这对理解和控制材质的视觉效果至关重要。

3. **`dot` 函数**

    点积。这种情况下，dot product 的结果是加权平均值。

4. **`0.3333333.xxx` 向量**

    - `0.3333333.xxx` 表示一个三维向量，其中每个分量均为 `0.3333333`。

        写为 `float3(0.3333333, 0.3333333, 0.3333333)` 更清晰

    - 这个向量用来对 `F0` 的 RGB 分量进行等权重的平均计算。`0.3333333` 是 `1/3` 的近似值，所以该操作实际上是取 RGB 三个通道的平均。

5. **函数返回值**

    `float` 类型的 `F0`，表示 RGB 三个通道的平均反射率。



### 5.3 F0RGBToDielectricSpecular(float3 F0)

```c++
float F0RGBToDielectricSpecular(float3 F0)
{
    return F0ToDielectricSpecular(FRGBToF0(F0));
}
```

不再解释



### 5.4 DielectricSpecularToF0

```c++
half DielectricSpecularToF0(half Specular)
{
    return half(0.08f * Specular);
}
```

`half` 是一种表示 16 位浮点数的类型，通常用于图形渲染中的 shader 编程。与 32 位的 `float` 相比，`half` 占用的**内存更少** (2 个字节 )，但是他的**精度和范围相应减少**。

- **`half` 的使用场景**
    - **性能优化**
        - 在 GPU 编程中，使用 `half` 类型可以节省内存和计算资源，特别是在需要处理大量数据或在移动设备上进行渲染时
    - **精度需求较低**
        - 对于一些不需要高精度的计算（如 <font color=red>**颜色值的计算、高光反射等**</font>），`half` 类型合适
    
- **为什么用 0.08**

    见上文 5.1



### 5.5 DielectricF0ToIor

```c++
// [Burley, "Extending the Disney BRDF to a BSDF with Integrated Subsurface Scattering"]
float DielectricF0ToIor(float F0)
{
    return 2.0f / (1.0f - sqrt(min(F0, 0.99))) - 1.0f;
}
```

$$
\frac{2.0f}{(1.0f - \sqrt{min(F_0, 0.99)})} - 1.0f
$$

- 为什么用上面的公式

    - Fresnel Equations 简化版本
        $$
        F_0 = \bigg(\frac{IOR - 1}{IOR + 1}\bigg)^2
        $$
        推导出
        $$
        IOR=\frac{1 + \sqrt{F_0}}{1-\sqrt{F_0}}
        $$

    - 理论上 $F_0$ 介于 [0, 1]之间。为防止出现极端值（如 $F_0 = 1$ 会出现错误），将 $F_0$ 限制在 0.99 以下

    - 上式和 代码中的公式是完全相等的

根据计算出来的折射率 Index of Refraction 可以直接用于 Shader 的计算，以决定光线在进入和离开材料时的折射和反射行为。



### 5.6 DielectricF0RGBToIor

```c++
float DielectricF0RGBToIor(float3 F0)
{
    return DielectricF0ToIor(F0RGBToF0(F0));
}
```



### 5.7 DielectricIorToF0

```c++
float DielectricIorToF0(float Ior)
{
    const float F0Sqrt = (Ior - 1) / (Ior + 1);
    const float F0 = F0Sqrt * F0Sqrt;
    return F0;
}
```

见 5.5



### 5.8 MicroOcclusion 微观遮蔽

- 代码

```c++
// Anything with Specular less than 2% is physically impossible and is instead considered to be shadowing.
float GetF0MicroOcclusionThreshold() { return 0.02f; }
float F0ToMicroOcclusion(float F0) { return saturate(50.0 * F0); }
float3 F0ToMicroOcclusion(float3 F0) { return saturate(50.0 * F0); }

float F0RGBToMicroOcclusion(float3 F0)
{
    return F0ToMicroOcclusion(max(F0.r, max(F0.g, F0.b)));
}
```

#### 解释每个部分：

1. **`GetF0MicroOcclusionThreshold()`**：

    ```cpp
    float GetF0MicroOcclusionThreshold() { return 0.02f; }
    ```

    这个函数返回一个阈值，表示当 `F0`（反射率）低于 2% 时，应该被视为微观遮蔽。这个值意味着如果 `F0` 低于 0.02，通常情况下这种情况被认为是微观遮蔽而不是实际的反射现象。

2. **`F0ToMicroOcclusion(float F0)`**：

    ```cpp
    float F0ToMicroOcclusion(float F0) { return saturate(50.0 * F0); }
    ```

    这个函数将单一的 `F0` 值（反射率）转换为微观遮蔽值。计算公式是将 `F0` 乘以 50，然后使用 `saturate` 函数确保结果在 0 到 1 的范围内（`saturate` 函数是一个将值限制在 0 到 1 之间的操作）。

3. **`F0ToMicroOcclusion(float3 F0)`**：

    ```cpp
    float3 F0ToMicroOcclusion(float3 F0) { return saturate(50.0 * F0); }
    ```

    这个函数处理一个 `float3` 类型的 `F0`，这通常表示颜色的 RGB 值。它将每个颜色分量的 `F0` 值乘以 50，然后使用 `saturate` 函数处理，确保结果在 0 到 1 的范围内。



#### 背景

- **F0** 是反射率的表示，通常在物理基础渲染中用于计算光的反射特性。它表示的是在正常视角下材料的反射率。
- **微观遮蔽** （Micro-Occlusion）指的是由于材料表面微小的粗糙度或不规则性，导致部分光线被遮挡或陷入这些微观结构中的现象。即使在看起来非常光滑的表面上，微观层面仍然存在小的凹凸不平，这些不规则性会影响光的反射和散射。



#### 微观遮蔽的作用

在物理基础渲染（PBR）中，光线与表面的相互作用不仅仅取决于表面的宏观特性（如光滑或粗糙），还取决于表面的微观结构。微观遮蔽是一个重要的现象，因为它影响了：

1. **反射强度**：表面上的微小凹凸可能会导致某些区域的光线被遮挡，减少反射强度，特别是在较低角度的反射情况下。
2. **漫反射**：这些微观细节也会增加漫反射的比例，因为更多的光线被散射而不是直接反射。



#### 为什么 F0 < 0.02 会被视为微观遮蔽？

在物理基础渲染中，`F0` 代表的是材料在法线方向上的反射率（即在垂直于表面方向上的反射强度）。当 `F0` 值非常低（如低于 0.02）时，几乎没有直接反射，这通常不再被视为真正的反射，而是光线被表面微观结构捕获或遮蔽的结果。这种情况下，光线更有可能被认为是受到了微观遮蔽，而不是像镜面一样的反射。

因此，当 `F0` 低于 0.02 时，Unreal Engine 会将其视为微观遮蔽的效果，而不是反射的结果。这是为了在渲染过程中模拟更真实的光与表面相互作用，确保材料在视觉上符合物理定律。



#### 微观遮蔽的计算

当视为微观遮蔽时，计算通常会通过调整光线的反射或散射模型来反映微观结构对光线的影响。主要考虑的是如何减少光的直接反射，并增加光的散射或吸收。以下是一些通常的方法：

##### 1. 减少反射强度

当 `F0` 值非常低时（如低于 0.02），直接反射的光线贡献会显著降低。这时，渲染引擎会减少光线的反射强度，以模拟光线被表面微观结构遮挡或陷入其中的效果。

在实际计算中，可以通过使用一个遮蔽函数，将反射强度乘以一个遮蔽系数。例如：

```cpp
float MicroOcclusion = saturate(50.0 * F0);
float Reflectance = BaseReflectance * (1.0 - MicroOcclusion);
```

这里的 `MicroOcclusion` 是一个基于 `F0` 计算的遮蔽值，`BaseReflectance` 是材料的基础反射率。最终的 `Reflectance` 会被 `MicroOcclusion` 减弱。

##### 2. **增加漫反射贡献

当光线被微观遮蔽时，表面的反射通常会减少，而漫反射成分可能会增加。漫反射是指光线在撞击表面后被多次散射，最终以更随机的方向离开表面。

这种情况下，渲染引擎可能会通过增加漫反射的比例来补偿减少的反射。例如：

```cpp
float DiffuseContribution = BaseDiffuse * MicroOcclusion;
```

在这种情况下，`MicroOcclusion` 值较高时会增加漫反射的贡献，以模拟光线被微观结构散射的效果。

##### 3. 调整光照模型

渲染引擎可能会调整使用的光照模型，以更好地反映表面微观遮蔽的影响。比如在 Cook-Torrance 模型中，通过使用遮蔽和阴影（Shadowing and Masking）函数来模拟微观表面结构对光线反射的影响。

具体的计算可能包括：

- Schlick’s approximation: 通过调整反射系数 `F0` 来近似计算遮蔽效果。
- Smith’s Masking Function: 这种函数用来模拟微观表面结构对光线遮蔽的影响，特别是在高光区域。

##### 4. 物理遮蔽与环境光遮蔽结合

微观遮蔽有时也会与环境光遮蔽（Ambient Occlusion, AO）结合。AO 是一种技术，用于模拟因为物体之间的接触或复杂几何形状导致的光线遮蔽。当表面微观遮蔽较强时，AO 的影响可能会被加大。

```cpp
float CombinedOcclusion = AO * MicroOcclusion;
```

这种组合可以模拟更加复杂的光遮蔽效果，提升渲染结果的真实感。



#### 总结

当光的反射被视为微观遮蔽时，通常会减少直接反射光的强度，增加漫反射的贡献，并通过调整光照模型来模拟表面的微观结构对光的影响。这种处理方法有助于更好地模拟真实世界中光线与复杂表面之间的相互作用。



### 5.9 <font color=red>ComputeF0</font>

```cpp
half3 ComputeF0(half Specular, half3 BaseColor, half Metallic)
{
    return lerp(DielectricSpecularToF0(Specular).xxx, BaseColor, Metallic.xxx);
}
```

这段代码是用于计算 F0 值（反射率）的一个函数。`F0` 决定了在法线方向上材料的反射特性。

- **参数**
    - `Specular`
        - `half` 类型浮点数，表示材料的高光强度或反射率。对于非金属材料，这个值通常是固定的，如 0.04，对于金属材料，`Specular` 会影响反射率。
    - `BaseColor`
        - `half3` 类型向量，表示材料的基础颜色。
        - 对于 **非金属材料**，通常代表 **漫反射** 颜色
        - 对于 **金属材料**，通常表示 **反射** 颜色
    - `Metallic`
        - `half` 类型浮点数，表示材料的金属度。`0 ~ 1` 之间，0 表示完全非金属，1 表示完全金属。
    
- **`Lerp(A, B, t)`**

    - `lerp` 线性插值函数

        - 根据 `t` 在 `A` 和 `B` 之间插值。当 `t=0`，返回 `A`；当 `t=1`，返回 `B`；当 `0 < t < 1`，返回

            $result = (1-t) \times A + t \times B$
            
            - 怎么求，y=ax + b, y=at+b, A=b, B=a * 1+b = a + A, a=B-A, 故 y=(B-A)t+A
        
    - 这段代码中
    
        ```c++
        lerp(DielectricSpecularToF0(Specular).xxx, BaseColor, Metallic.xxx)
        ```
    
        是根据 金属度 在 根据电介质高光系数算出的 F0 和 基础颜色 中进行插值。
    
        如果 `Metallic` 为 0（完全非金属），则返回 电介质高光系数对应的 F0
    
        如果 `Metallic` 为 1 （完全金属），则返回 `BaseColor` 作为 反射率 F0
    
        对于部分金属，返回插值。

- **总结**
  - **非金属材料**
    - 对于非金属材料`Metallic = 0`
      - `F0` 由 `DielectricSpecularToF0(Specular)` 决定，通常这个值是固定的（如 0.04），表示非金属在法线方向的反射率
  - **金属材料**
    - `Metallic = 1`
      - `F0` 由 `BaseColor` 决定，因为金属材料的颜色直接影响其反射率
  - **混合材料**
    - `0 < Metallic < 1`
      - `F0`是非金属反射率与基色之间的插值，具有部分金属特性



### 5.10 ComputeF90

```c++
float3 ComputeF90(float3 F0, float3 EdgeColor, float Metallic)
{
    return lerp(1.0, EdgeColor, Metallic.xxx);
}
```

- `EdgeColor`
    - `float3` 类型，表示材料在高角度（接近 90° 观察角）下的反射率颜色
    - 对于 **金属** 材料，`EdgeColor` 通常会与材料的 **基色** 相匹配，因为金属材料在不同角度下的反射颜色是由其本身的物理特性决定的
    - 对于 **非金属** 材料，`EdgeColor` 通常接近白色（即 `float3(1.0, 1.0, 1.0)`），因为 **大多数非金属材料在极端角度下的反射率趋向于 100%**

- `Metallic`
    - `float` 类型，表示材料的金属度，范围在 [0, 1] 之间
    - `Matallic = 0` 表示材料是完全非金属的
    - `Matallic = 1` 表示材料是完全金属的
    - `Matallic.xxx` 将这个标量扩展到 `float3` ，用于与颜色值插值

- **核心逻辑**
    - `lerp(1.0, EdgeColor, Metallic.xxx)`
        - lerp 线性插值函数
        - 当 `Metallic` 为 0 时，return 1.0 (代表非金属材料的 F90 接近 **白色**，即高光区域反射率为 100%)
        - 当 `Metallic` 为 1 时，return `EdgeColor`，表示金属材料的 F90 与其 basecolor 一致
        - 其它 return $(EdgeColor - 1.0) * Metallic.xxx + 1.0$



- **EdgeColor 的作用**
    - `EdgeColor` 用于 <font color=red>控制金属材料在 **高角度** 下的反射颜色</font>。这是因为金属材料的反射光线不仅仅是镜面反射，还和材料的本身颜色有关。对于非金属材料，`EdgeColor` 通常不被使用，反射率趋于白色。



### 5.11 ComputeDiffuseAlbedo

```c++
float3 ComputeDiffuseAlbedo(float3 BaseColor, float Metallic)
{
    return BaseColor - BaseColor * Metallic;
}
```

> - albedo 反射率

通过减去 `BaseColor` 与 `Metallic` 成比例的部分，来调整漫反射反照率。

- `Metallic = 0`，结果是 `BaseColor`，即 **非金属材料保留其全部 basecolor 作为漫反射反照率**
- `Metallic = 1`，结果是 `float3(0.0, 0.0, 0.0)` ，即 <font color=red>完全金属材料没有漫反射颜色，因为金属材料的反照率 albedo 完全由镜面反射 （高光 `Specular`） 决定，而不是由漫反射决定</font>

- `0 < Metallic < 1` ，插值，表示部分金属材料具有部分漫反射特性。



### 5.12 MakeRoughnessSafe

```c++
float MakeRoughnessSafe(float Roughness, float MinRoughness = 0.001f)
{
    return clamp(Roughness, MinRoughness, 1.0f);
}
```

用于确保 **粗糙度在一个合理的范围内**，防止粗糙度过小导致渲染问题。

- 高粗糙度
    - 表面更加粗糙，反射更模糊
- 低粗糙度
    - 表面更加光滑，反射更加清晰

`MinRoughness = 0.001f` 防止粗糙度过小，导致物理不合理的镜面反射



- `clamp` 函数

    - `clamp(a, b, c)`

        - 将 a 限制在 `[b, c]` 范围内。

            ```c++
            if (a < b) { return b; } 
            else if (a > c) { return c; } 
            else { return a; }
            ```

            

### 5.13 F0ToMetallic

```c++
float F0ToMetallic(float F0)
{
    // Approximate the metallic input from F0 with a small lerp region
    
    // Instead of DiamondF0 = 0.24, the metallic region starts right after metallic > 0 and specular=1 to match legacy.
    const float FullMetalBeginF0 = 0.08f;
    // roughly the end of semi-conductor
    const float FullMetalEndF0 = 0.4f;
    // This is compatible with UE shading model mapping allowing F0 to take a value up to 0.08 for dielectric.
    
    return saturate((F0 - FullMetalBeginF0) / (FullMetalEndF0 - FullMetalBeginF0));
}
```

- **参数说明**

    - `FullMetalBeginF0`
        - `0.08f`，表示金属区域开始的 F0 值
        - 当 F0 值大于 0.08 时，认为材料开始表现出金属特性
    - `FullMetalEndF0`
        - `0.4f`，表示完全金属的 F0 值
        - 当 F0 值达到 0.4 时，材料被视为完全金属

- **核心逻辑**

    - $$
        \frac{F0 - FullMetalBeginF0}{FullMetalEndF0 - FullMetalBeginF0}
        $$

        - 这个表达式将 F0 从 `[FullMetalBeginF0, FullMetalEndF0]` 的范围 **线性映射** 到 `[0, 1]` 的范围
        - 当 `F0 = FullMetalBeginF0` 即 0.08 时，结果为 0，表示材料没有金属特性
        - 当 `F0 = FullMetalEndF0` 即 0.4 时，结果为 1，表示材料是完全金属的

    - `saturate()`

        - 将结果限制在 `[0, 1]` 之间，确保金属度不会超出合理范围

> - **结合注释解释一下为什么是 0.08 和 0.4**
>
> 这段代码中的两个常量 `0.08f` 和 `0.4f` 代表了在反射率（F0）和金属度（Metallic）之间的一个转换区间，具体选择这些值与实际材料的物理属性以及渲染模型的设计有关。
>
> ### 注释分析与解释
>
> ```cpp
> // Approximate the metallic input from F0 with a small lerp region
> // Instead of DiamondF0 = 0.24, the metallic region starts right after metallic > 0 and specular=1 to match legacy.
> const float FullMetalBeginF0 = 0.08f;
> // roughly the end of semi-conductor
> const float FullMetalEndF0 = 0.4f;
> // This is compatible with UE shading model mapping allowing F0 to take a value up to 0.08 for dielectric.
> ```
>
> #### 为什么是 `0.08f`？
>
> - **`FullMetalBeginF0 = 0.08f`**:
>     - 这个值表示金属区域开始的 F0 值，也就是当 `F0` 大于 `0.08` 时，材料开始表现出金属特性。
>     - 原因：
>         - `0.08` 是一个经验值，通常用于表示非金属材料（如塑料、陶瓷、木材等）的最大反射率。非金属材料的 F0 通常处于较低范围，一般在 0.02 到 0.08 之间。
>         - 在传统渲染模型中，非金属的 F0 被限制在 0.08 以下，因此 `0.08` 作为金属特性开始的阈值，是为了确保材料在低 F0 时表现为非金属。
> - **与 `DiamondF0 = 0.24` 的比较**：
>     - 注释中提到，Diamond（钻石）的 F0 值为 `0.24`，这是一个介于完全非金属和完全金属之间的材料的反射率。
>     - 选择 `0.08` 作为金属区域的开始，而不是使用 `0.24`，是为了与传统的材质模型保持一致，即在低 F0 范围内，不应显示出金属特性。
>
> #### 为什么是 `0.4f`？
>
> - `FullMetalEndF0 = 0.4f`:
>     - 这个值表示完全金属材料的 F0 值，也就是当 `F0` 达到 `0.4` 时，材料被视为完全金属。
>     - 原因：
>         - `0.4` 是一个接近典型金属材料（如铝、银等）的反射率。金属材料的 F0 通常较高，在 0.4 到 0.9 之间，但一般用 `0.4` 作为金属特性的下限是因为大多数金属在 0.4 处已经开始表现出显著的镜面反射特性。
>         - 这个值也考虑了半导体材料的特性，这些材料的 F0 值通常在 0.2 到 0.4 之间，表示从非金属到金属之间的过渡。
>
> #### 渲染模型的兼容性
>
> - 兼容性：
>     - 注释最后提到，这个设计与 Unreal Engine 的着色模型兼容，使得 F0 的值可以在 0.08 以下表示电介质（非金属）材料。
>     - 这确保了在转换 F0 到金属度的过程中，低 F0 值（非金属）不会错误地表现为金属特性，而较高的 F0 值则能够正确地映射为金属。
>
> ### 总结
>
> - `0.08` 和 `0.4`
>
>      作为阈值选择，是基于材料的物理特性和渲染模型的经验参数：
>
>     - `0.08` 确保低反射率材料保持非金属属性。
>     - `0.4` 确保高反射率材料被正确识别为金属。
>     - 这两个值之间的区间则用于表示部分金属特性，使得材料能够在渲染过程中平滑过渡。



### 5.14 F0RGBToMetallic

```c++
float F0RGBToMetallic(float3 F0)
{
    return F0ToMetallic(max(F0.r, max(F0.g, F0.b)));
}
```

选用 `F0` 各通道中的最大值来进行转换。

- 为什么选用最大值？
    - 在实际应用中，不同材料可能在不同颜色通道上有不同的反射率。例如，金属材料可能在一个通道上有较高的反射率，而在另一个通道上较低。
    - 选择最大的反射率值来确定材料的金属度可以确保 **材料中最亮的部分决定其金属特性**。



## 6. Layering 分层

这部分处理材质的垂直分层效果，例如某些透明或者半透明材质可能有多层次结构

### 6.1 结构体 FVerticalLayeringInfo

```c++
// Layering, coverage and transmittance

struct FVerticalLayeringInfo
{
    float TransmittanceTopAndBottom;	// Ratio of transmittance of both and bottom surface only
    float TransmittanceOnlyBottom;		// Ratio of transmittance of bottom surface only
    float TransmittanceOnlyTop;			// Ratio of transmittance of top surface only
    
    float SurfaceBottom;				// Ratio of bottom surface visible
    float SurfaceTop;					// Ratio of top surface visible
    
    float Coverage;						// Ratio of any surface is visible
    float NoSurface;					// Ratio of no surface is visible
};

// This function returns surface information after a layering of two slab of matter having difference and uncorelated coverate.
```

结构体 `FVerticalLayeringInfo`，存储了与两层材质叠加有关的各种信息：

1. `TransmittanceTopAndBottom`:  顶层和底层同时透射的比例
2. `TransmittanceOnlyBottom`: 只有底层投射的比例（顶层不透射）
3. `TransmittanceOnlyTop`: 只有顶层透射的比例（底层不透射）
4. `SurfaceBottom`: 底层表面可见的比例
5. `SurfaceTop`: 顶层表面可见的比例
6. `Coverage`: 任何表面可见的比例（顶层或底层或两者都可见）
7. `NoSurface`: 没有任何表面可见的比例



### 6.2 函数 GetVerticalLayeringInfo

```c++
// This function returns surface information after a layering of two slab of matter having difference and uncorelated coverage.
// See https://www.desmos.com/calculator/qg5yc3nnr4
FVerticalLayeringInfo GetVerticalLayeringInfo(const float TopCoverage, const float BottomCoverage)
{
    FVerticalLayeringInfo Info;
    
    Info.TransmittanceTopAndBottom	= TopCoverage * BottomCoverage;
    Info.TransmittanceOnlyBottom	= (1.0f - TopCoverage) * BottomCoverage;
    Info.TransmittanceOnlyTop		= (1.0f - BottomCoverage) * TopCoverage;
    
    Info.SurfaceBottom				= Info.TransmittanceOnlyBottom;
    // Info.SurfaceTop = Info.TransmittanceOnlyTop + Info.TransmittanceTopAndBottom
    Info.SurfaceTop					= TopCoverage;
    
    // == Info.TransmittanceTopAndBottom + Info.TransmittanceOnlyBottom + Info.TransmittanceOnlyTop;
    // == Info.TopCoverage + Info.TransmittanceOnlyBottom
    Info.Coverage					= Info.SurfaceTop + Info.SurfaceBottom;
    Info.NoSurface					= 1.0 - Info.Coverage;
    
    return Info;
}
```

- **参数**
    - `TopCoverage`
        - 顶层材料的覆盖度（即材料占据空间的比例）
    - `BottomCoverage`
        - 底层材料的覆盖度
- **返回**
    - `FVerticalLayerInfo` 结构体，包含了关于两层材料叠加后的各种可见性和透射信息

#### 详细分析

1. **计算重叠透射（TransmittanceTopAndBottom）**

    ```cpp
    Info.TransmittanceTopAndBottom = TopCoverage * BottomCoverage;
    ```

    - 这是计算顶层和底层材料同时透射的比例，即两层材料的重叠部分。这部分比例由顶层覆盖度与底层覆盖度相乘得到。

2. **计算只有底层透射的比例（TransmittanceOnlyBottom）**

    ```cpp
    Info.TransmittanceOnlyBottom = (1.0f - TopCoverage) * BottomCoverage;
    ```

    - 这里考虑的是只有底层透射而顶层不透射的情况。通过 `(1.0f - TopCoverage)` 表示顶层不覆盖的比例，然后乘以底层的覆盖度得到该比例。

3. **计算只有顶层透射的比例（TransmittanceOnlyTop）**

    ```cpp
    Info.TransmittanceOnlyTop = (1.0f - BottomCoverage) * TopCoverage;
    ```

    - 这一行类似于上面一行的逻辑，只是这次考虑的是只有顶层透射而底层不透射的情况。通过 `(1.0f - BottomCoverage)` 表示底层不覆盖的比例，然后乘以顶层的覆盖度得到该比例。

4. **计算底层表面可见性（SurfaceBottom）**

    ```cpp
    Info.SurfaceBottom = Info.TransmittanceOnlyBottom;
    ```

    - 底层表面可见性与 `TransmittanceOnlyBottom` 一致，因为当底层透射但顶层不透射时，底层就是可见的。

5. **计算顶层表面可见性（SurfaceTop）**

    ```cpp
    Info.SurfaceTop = TopCoverage;
    ```

    - 顶层表面可见性直接等于顶层的覆盖度，因为顶层的可见性不仅包括它自己透射的部分，还包括透射且重叠的部分。

6. **计算总覆盖度（Coverage）**

    ```cpp
    Info.Coverage = Info.SurfaceTop + Info.SurfaceBottom;
    ```

    - 总覆盖度表示的是顶层和底层的所有可见部分的总和。由于两个表面可能部分重叠，因此需要将可见部分相加。

7. **计算没有表面可见的比例（NoSurface）**

    ```cpp
    Info.NoSurface = 1.0f - Info.Coverage;
    ```

    - 这是反映没有任何表面可见的比例，通过从 1.0（表示全部覆盖）中减去总覆盖度来得到。

#### 代码整体逻辑

- 可用于模拟现实世界中具有 **层次结构的材料**，如 <font color=red>**半透明材质、植被等，计算它们在不同覆盖情况下的光线透射和遮挡效果**</font>



## 7. 总结

`ShadingCommon.ush` 涉及了多个与材质渲染相关的基础工具和函数，如 

- ShadingModel 的定义和管理
- 材质参数的计算，主要是 `F0`, `IOR`, `F90` 等等之间的换算等
- 层次化材质的处理

这些工具和函数帮助 UE 实现了复杂的材质渲染效果，确保在不同平台和条件下的渲染一致性。









