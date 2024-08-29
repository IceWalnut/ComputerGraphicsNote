# UE 魔改引擎 1

视频：https://www.bilibili.com/video/BV1P14y1a7zj

文章：https://www.bilibili.com/read/cv18792540/



## C++

### 1. EngineTypes.h

修改 `EMaterialShadingModel` 这个枚举，加上 `QToon` 着色器

```c++
MSM_QToon					UMETA(DisplayName="QToon"),
```



### 2. MaterialShader.cpp

`GetShadingModelString` 方法中

```c++
case MSM_QToon:				ShadingModelName = TEXT("MSM_QToon"); break;
```



### 3. MaterialHLSLEmitter.cpp

影响到生成 Shader 的宏

```c++
if (ShadingModels.HasShadingModel(MSM_QToon))
		{
			OutEnvironment.SetDefine(TEXT("MATERIAL_SHADINGMODEL_QTOON"), TEXT("1"));
			NumSetMaterials++;
		}
```



### 4. HLSLMaterialTranslator.cpp

在 `GetMaterialEnvironment` 函数中

```c++
if (ShadingModels.HasShadingModel(MSM_QToon))
		{
			OutEnvironment.SetDefine(TEXT("MATERIAL_SHADINGMODEL_QTOON"), TEXT("1"));
			NumSetMaterials++;
		}
```



### 5. Material.cpp

`IsPropertyActive_Internal` 函数中，修改次表面接口和自定义接口
这里修改主要是为了第一次就编译完，不要等到后面再修改，编译引擎会很慢

```c++
// MP 应该是 MaterialProperty
case MP_CustomData0:
	Active = ShadingModels.HasAnyShadingModel({ MSM_ClearCoat, MSM_PreintegratedSkin, MSM_TwoSidedFoliage, MSM_Cloth, MSM_SubsurfaceProfile, MSM_QToon });
	break;
case MP_CustomData1:
	Active = ShadingModels.HasAnyShadingModel({ MSM_ClearCoat, MSM_Hair, MSM_Eye, MSM_SubsurfaceProfile, MSM_QToon });
	break;
```



### 6. MaterialShared.h/cpp

次表面接口添加我们的 shader

判断是否是 `SubsurfaceShadingModel` 以传入 `subsurface` 的接口数据

```c++
inline bool IsSubsurfaceShadingModel(FMaterialShadingModelField ShadingModel)
{
    return ShadingModel.HasShadingModel(MSM_Subsurface) || ShadingModel.HasShadingModel(MSM_PreintegratedSkin) ||         ShadingModel.HasShadingModel(MSM_SubsurfaceProfile) || ShadingModel.HasShadingModel(MSM_TwoSidedFoliage) || ShadingModel.HasShadingModel(MSM_Cloth) || ShadingModel.HasShadingModel(MSM_Eye) || ShadingModel.HasShadingModel(MSM_QToon);
}
```



在 `MaterialAttributeDefinitionMap.cpp` 中的  `FMaterialAttributeDefinitionMap::GetAttributeOverrideForMaterial` 修改 `CustomData0` 接口名称

```c++
case MP_SubsurfaceColor:
		if (Material->MaterialDomain == MD_Volume)
		{
			return LOCTEXT("Extinction", "Extinction");
		}
		CustomPinNames.Add({ MSM_Cloth, "Fuzz Color" });
		// TODO: fix begin, QToon // 6
		CustomPinNames.Add({ MSM_QToon, "QToonSubsurfaceColor" });
		// TODO: fix end
		return FText::FromString(GetPinNameFromShadingModelField(Material->GetShadingModels(), CustomPinNames, "Subsurface Color"));

case MP_CustomData0:
		CustomPinNames.Add({ MSM_ClearCoat, "Clear Coat" });
		CustomPinNames.Add({ MSM_Hair, "Backlit" });
		CustomPinNames.Add({ MSM_Cloth, "Cloth" });
		CustomPinNames.Add({ MSM_Eye, "Iris Mask" });
		CustomPinNames.Add({ MSM_SubsurfaceProfile, "Curvature" });
		// TODO: fix begin, QToon // 6
		CustomPinNames.Add({ MSM_QToon, "Range" });
		// TODO: fix end
		return FText::FromString(GetPinNameFromShadingModelField(Material->GetShadingModels(), CustomPinNames, "Custom Data 0"));
```



### 7. ShaderMaterial.h

接口开放后，要启用 CustomData GBuffer，先在 `ShaderMaterial.h` 中的  `FShaderMaterialPropertyDefines` 结构体中添加相关位域：

```c++
// TODO: fix begin, QToon // 7
	uint8 MATERIAL_SHADINGMODEL_QTOON : 1;
	// TODO: fix end
```



### 8. ShaderMaterialDerivedHelpers.cpp

开启 `CustomData GBuffer` 写入

在 `Dst.WRITES_CUSTOMDATA_TO_GBUFFER` 最后多加一个 `Mat.MATERIAL_SHADINGMODEL_QTOON` 的判断

```c++
// TODO: fix begin, QToon // 8
	Dst.WRITES_CUSTOMDATA_TO_GBUFFER = (Dst.USES_GBUFFER && (Mat.MATERIAL_SHADINGMODEL_SUBSURFACE || Mat.MATERIAL_SHADINGMODEL_PREINTEGRATED_SKIN || Mat.MATERIAL_SHADINGMODEL_SUBSURFACE_PROFILE || Mat.MATERIAL_SHADINGMODEL_CLEAR_COAT || Mat.MATERIAL_SHADINGMODEL_TWOSIDED_FOLIAGE || Mat.MATERIAL_SHADINGMODEL_HAIR || Mat.MATERIAL_SHADINGMODEL_CLOTH || Mat.MATERIAL_SHADINGMODEL_EYE || Mat.MATERIAL_SHADINGMODEL_QTOON));
	// TODO: fix end
```



### 9. ShaderGenerationUtil.cpp

1.  在 `FShaderCompileUtilities::ApplyFetchEnvironment` 中添加

    ```c++
    // TODO: fix begin, QToon // 9
    	FETCH_COMPILE_BOOL(MATERIAL_SHADINGMODEL_QTOON);
    	// TODO: fix end
    ```

    

2. 在 `DeterminUsedMaterialSlots` 方法中添加判断

    ```c++
    // TODO: fix begin, QToon // 9
    	if (Mat.MATERIAL_SHADINGMODEL_QTOON)
    	{
    		SetStandardGBufferSlots(Slots, bWriteEmissive, bHasTangent, bHasVelocity, bHasStaticLighting, bIsStrataMaterial);
    		Slots[GBS_CustomData] = bUseCustomData;
    	}
    	// TODO: fix end
    ```

    

C++ 部分就结束了



## Shader

UE4 的材质系统只保留了两个 Custom 接口

UE4 的延迟渲染管线只有一个自定义的 GBuffer 允许开发者自由使用

我们要把 **次表面接口** 也用上，就可以省一个接口，供其他数据使用

1. 编辑 `ConsoleVariables.ini`，将 `r.ShaderDevelopmentMode=1` 打开
2. 编译 shader 时可以使用 `Ctrl + Shift + .` ，`.` 是大键盘的 `.` 键

- 须知：
    - 编译 shader 是在 UE 中进行的，C++ 改完后修改 `.ush/usf` 文件只需要进入 UE 随时使用 `Ctrl + Shift + .` 即可编译

![image-20240809214630267](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408092146349.png)

这行原来是注释掉的

Shader 需要做的前置工作有注册我们的 `QToon`



### 1. ShadingCommon.ush

**注册工作**

![image-20240809215333645](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408092153683.png)







### 2. Definitions.usf

**注册工作**

```c++
// TODO: fix begin, QToon // shader 2
#ifndef MATERIAL_SHADINGMODEL_QTOON
#define MATERIAL_SHADINGMODEL_QTOON
#endif
// TODO: fix end
```





### 3. BasePassPixelShader.usf

首先看 `BasePassPixelShader` 做了什么

> 在 Unreal Engine (UE) 中，`BasePassPixelShader.usf` 是一个非常重要的着色器文件。它主要负责处理每个像素的基本渲染过程（Base Pass），并为后续的渲染阶段准备必要的数据。这个着色器在渲染流程中有几个主要作用：
>
> ### 1. **处理材质和光照模型**
>
> `BasePassPixelShader.usf` 负责将材质的所有特性（如漫反射、法线、金属度、粗糙度等）与场景中的光照信息结合在一起，从而计算出每个像素的最终颜色值。它会采样各种材质贴图（如漫反射、法线、RMO 等），并根据光照模型（例如 Lambert、Phong、PBR 等）计算出光照对像素颜色的影响。
>
> ### 2. **生成GBuffer**
>
> 在延迟渲染（Deferred Rendering）模式下，`BasePassPixelShader.usf` 的输出会被存储在多个 GBuffer 中。这些 GBuffer 包含了物体表面的各种信息，如世界空间的法线、材质属性（如金属度、粗糙度）、漫反射颜色等。后续的渲染阶段会使用这些信息来处理全局光照、反射等效果。
>
> ### 3. **阴影计算**
>
> 在某些情况下，`BasePassPixelShader.usf` 还负责计算阴影效果。它会考虑光源投射的阴影，以及物体本身的自阴影，来决定每个像素的阴影强度。
>
> ### 4. **透明物体的处理**
>
> 虽然延迟渲染通常不处理透明物体，但在前向渲染（Forward Rendering）或混合模式下，`BasePassPixelShader.usf` 可能会处理透明材质的着色。它会根据透明度来混合背景和前景颜色，从而生成正确的透明效果。
>
> ### 参与渲染流程的时机
>
> `BasePassPixelShader.usf` 在渲染流程中的位置主要是在 Base Pass 阶段。Base Pass 是整个渲染管线的早期阶段，它的作用是为场景中的所有几何体生成基本的像素颜色和其他需要的渲染数据。具体来说，Base Pass 通常是渲染每个几何体的第一次通道，因此它也是渲染过程中比较耗时的阶段之一。
>
> ### 典型的渲染流程简述
>
> 1. **几何阶段（Geometry Phase）**：这里是进行顶点处理，计算顶点位置和其他顶点属性的阶段。
> 2. **Base Pass**：`BasePassPixelShader.usf` 在这里起作用。它计算每个像素的材质属性和光照效果，并输出到 GBuffer。
> 3. **光照阶段（Lighting Phase）**：使用 GBuffer 中存储的信息计算最终的光照效果（包括全局光照、阴影、反射等）。
> 4. **后处理阶段（Post-Processing Phase）**：对已经渲染好的图像应用各种效果，比如颜色校正、模糊、景深等。
>
> 通过这些处理，`BasePassPixelShader.usf` 确保每个像素在经过光照和材质计算后，具有正确的视觉表现。

这里的修改：

找到 `FPixelShaderInOut_MainPS` 函数添加判断条件，为 `SubsurfaceColor` 写入数据，不然 Subsurface 接口数据为空

```c++
// TODO: fix begin, QToon // shader 3
#if MATERIAL_SHADINGMODEL_SUBSURFACE || MATERIAL_SHADINGMODEL_PREINTEGRATED_SKIN || MATERIAL_SHADINGMODEL_SUBSURFACE_PROFILE || MATERIAL_SHADINGMODEL_TWOSIDED_FOLIAGE || MATERIAL_SHADINGMODEL_CLOTH || MATERIAL_SHADINGMODEL_EYE || MATERIAL_SHADINGMODEL_QTOON
	if (ShadingModel == SHADINGMODELID_SUBSURFACE || ShadingModel == SHADINGMODELID_PREINTEGRATED_SKIN || ShadingModel == SHADINGMODELID_SUBSURFACE_PROFILE || ShadingModel == SHADINGMODELID_TWOSIDED_FOLIAGE || ShadingModel == SHADINGMODELID_CLOTH || ShadingModel == SHADINGMODELID_EYE || ShadingModel == SHADINGMODELID_QTOON
	// TODO: fix end
        
        
        ...
        
// TODO: fix begin, QToon // shader 3
#if MATERIAL_SHADINGMODEL_CLOTH || MATERIAL_SHADINGMODEL_QTOON
		else if (ShadingModel == SHADINGMODELID_CLOTH || Shadingmodel == SHADINGMODELID_QTOON
		// TODO: fix end
                 ...
```



### 4. ShadingModelsMaterial.ush

在 `#if MATERIAL_SHADINGMODEL_EYE ***** #endif` 后面添加我们的 `QTOON` 宏判断，

将我们的从 **次表面** 接口的数据 和 **自定义接口的数据** 传入 `GBuffer.CustomData`

![image-20240819221408230](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408192214275.png)

```c++
	// TODO: fix begin, QToon // shader 4
#if MATERIAL_SHADINGMODEL_QTOON
	else if (ShadingModel ==  SHADINGMODELID_QTOON)
	{
		GBuffer.CustomData.rgb = EncodeSubsurfaceColor(SubsurfaceColor);
		GBuffer.CustomData.a = saturate( GetMaterialCustomData0(MaterialParameters) );
	}
#endif
	// TODO: fix end
```

- GBuffer 

  - 在 `DeferredShadingCommon.ush` 中

    ![](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408192216485.png)

  - `FGBufferData`

    ```c++
    // all values that are output by the forward rendering pass 
    struct FGBufferData
    {
        // normalized
        half3 WorldNormal;
        // normalized, only valid if HAS_ANOSOTROPY_MASK in SlectiveOutputMask
        half3 WorldTangent;
        // 0..1 (derived from BaseColor, Matalness, Specular)
        half3 DiffuseColor;
        // 0..1 (derived from BaseColor, Matalness, Specular)
        half3 SpecularColor;
        // 0..1, white for SHADINGMODELID_SUBSURFACE_PROFILE and SHADINGMODELID_EYE (apply BaseColor after scattering is more correct and less blurry)
        half3 BaseColor;
        // 0..1
        half Metallic;
        // 0..1
        half Specular;
        // 0..1
        half4 CustomData;
        // AO utility value
        half Generic AO;
        // Indirect irradiance luma
        half IndirectIrradiance;
        // Static shadow factors for channels assigned by Lightmass
        // Lights using static shadowing will pick up the appropriate channel in their deferred pass
        half4 PrecomputedShadowFactors;
        // 0..1
        half Roughness;
        // -1..1, only valid of only valid if HAS_ANISTROPY_MASK in SelectiveOutputMask
        half Anisotropy;
        // 0..1 ambient occlusion  e.g. SSAO, wet surface mask, skylight mask, ...
        half GBufferAO;
        // 漫反射非直接采样遮蔽的位掩码
        // Bit mask for occlusion of the diffuse indirect samples
        uint DiffuseIndirectSampleOcclusion;
        // 0..255
        uint ShadingModelID;
        // 0..255
        uint SelectiveOutputMask;
        // 0..1, 2 bits, use CastContactShadow(GBuffer) or HasDynamicIndirectShadowCasterRepresentation(GBuffer) to extract
        half PreObjectGBufferData;
        // in world uints
        half CustomDepth;
        // Custom depth stencil value
        uint CustomStencil;
        // in unreal units (linear), can be used to reconstruct world position
        // only valid when decoding the GBuffer as the value gets reconstructed from the Z buffer
        half Depth;
        // Velocity for motion blur (only used when WRITES_VELOCITY_TO_GBUFFER is enabled)
        half4 Velocity;
        
        // 0..1, only needed by SHADINGMODELID_SUBSURFACE_PROFILE and SHADINGMODELID_EYE which apply BaseColor later
        half3 StoredBaseColor;
        // 0..1, only needed by SHADINGMODELID_SUBSURFACE_PROFILE and SHADINGMODELID_EYE which apply Specular later
        half StoredSpecular;
        // 0..1, only needed by SHADINGMODELID_EYE which encodes Iris Distance inside Metallic
        half StoredMetallic;
        
        // Curvature for mobile subsurface profile
        half Curvature;
    }
    ```
    
    > 以下是 ChatGPT 给的解释
    >
    > 这段结构体定义了一个名为 `FGBufferData` 的结构体，它包含了在前向渲染（Forward Rendering）过程中输出的所有渲染相关数据。每个字段都代表在渲染过程中计算的特定信息，通常与光照、材质属性、阴影等相关。以下是对每个字段的详细解释：
    >
    > 1. **`WorldNormal` (`half3`)**: 物体表面的法线向量，已经归一化。用于计算光照方向和表面方向的关系。
    >
    > 2. **`WorldTangent` (`half3`)**: 物体表面的切线向量，已经归一化。只有在 `SelectiveOutputMask` 中启用了 `HAS_ANISOTROPY_MASK` 时才有效，用于各向异性材质的光照计算。
    >
    > 3. **`DiffuseColor` (`half3`)**: 表面反射的漫射颜色，值范围在 0 到 1 之间。该值是从材质的基色（BaseColor）、金属度（Metalness）和高光反射率（Specular）计算得到的。
    >
    > 4. **`SpecularColor` (`half3`)**: 表面反射的高光颜色，值范围在 0 到 1 之间。该值同样是从材质的基色（BaseColor）、金属度（Metalness）和高光反射率（Specular）计算得到的。
    >
    > 5. **`BaseColor` (`half3`)**: 表面的基色（颜色），值范围在 0 到 1 之间。对于次表面散射和眼睛的着色模型，白色表示在散射后应用基色更为正确且不那么模糊。
    >
    > 6. **`Metallic` (`half`)**: 表面的金属度，值范围在 0 到 1 之间。用于区分金属和非金属材质。
    >
    > 7. **`Specular` (`half`)**: 表面的高光反射率，值范围在 0 到 1 之间。
    >
    > 8. **`CustomData` (`half4`)**: 用于存储自定义数据的四维向量。可以用作材质的附加信息。
    >
    > 9. **`GenericAO` (`half`)**: 环境光遮蔽（AO）的通用值，用于模拟物体之间的遮挡效果。
    >
    > 10. **`IndirectIrradiance` (`half`)**: 间接光照的亮度，用于计算全局光照效果。
    >
    > 11. **`PrecomputedShadowFactors` (`half4`)**: 预计算的阴影因子，由 Lightmass 分配给不同的通道。在延迟渲染阶段，使用静态阴影的光源会选择相应的通道。
    >
    > 12. **`Roughness` (`half`)**: 表面的粗糙度，值范围在 0 到 1 之间。粗糙度影响光线在表面上的散射程度。
    >
    > 13. **`Anisotropy` (`half`)**: 各向异性材质的参数，值范围在 -1 到 1 之间，只有在 `SelectiveOutputMask` 中启用了 `HAS_ANISOTROPY_MASK` 时才有效。
    >
    > 14. **`GBufferAO` (`half`)**: 用于存储环境光遮蔽（AO）的值，如屏幕空间环境光遮蔽（SSAO）等。
    >
    > 15. **`DiffuseIndirectSampleOcclusion` (`uint`)**: 用于存储漫反射间接光照样本的遮挡信息的位掩码。
    >
    > 16. **`ShadingModelID` (`uint`)**: 着色模型的 ID，用于区分不同的着色模型。
    >
    > 17. **`SelectiveOutputMask` (`uint`)**: 选择性输出的掩码，用于控制哪些数据被写入 GBuffer。
    >
    > 18. **`PerObjectGBufferData` (`half`)**: 每个对象的 GBuffer 数据，值范围在 0 到 1 之间。使用 `CastContactShadow(GBuffer)` 或 `HasDynamicIndirectShadowCasterRepresentation(GBuffer)` 提取。
    >
    > 19. **`CustomDepth` (`half`)**: 自定义深度值，以世界单位为单位，通常用于渲染特效或自定义遮挡。
    >
    > 20. **`CustomStencil` (`uint`)**: 自定义的深度模板值，用于特定渲染通道或效果。
    >
    > 21. **`Depth` (`half`)**: 深度值，以 Unreal 单位为单位，可以用于重建世界位置。仅在解码 GBuffer 时有效，因为该值是从 Z 缓冲区中重建的。
    >
    > 22. **`Velocity` (`half4`)**: 用于运动模糊的速度数据，只有在启用了 `WRITES_VELOCITY_TO_GBUFFER` 时才会使用。
    >
    > 23. **`StoredBaseColor` (`half3`)**: 存储的基色值，仅在次表面散射和眼睛着色模型中需要，因为基色在应用散射后才被使用。
    >
    > 24. **`StoredSpecular` (`half`)**: 存储的高光反射率值，仅在次表面散射和眼睛着色模型中需要，因为高光反射率在应用散射后才被使用。
    >
    > 25. **`StoredMetallic` (`half`)**: 存储的金属度值，仅在眼睛着色模型中需要，该值还编码了虹膜距离。
    >
    > 26. **`Curvature` (`half`)**: 用于移动端次表面散射模型的曲率值。
    >
    > 这个结构体的主要作用是存储前向渲染过程中计算的各种光照和材质信息，以便后续的渲染步骤能够访问和使用这些数据。
  
  

### 5. BasePassCommon.ush

接口打开了，想要写入 `GBuffer.CustomData` 就要开启 GBuffer 的 CustomData 写入权限

```c++
// Only some shader models actually need custom data.
// TODO: fix begin, QToon // shader 5
#define WRITES_CUSTOMDATA_TO_GBUFFER		(USES_GBUFFER && (MATERIAL_SHADINGMODEL_SUBSURFACE || MATERIAL_SHADINGMODEL_PREINTEGRATED_SKIN || MATERIAL_SHADINGMODEL_SUBSURFACE_PROFILE || MATERIAL_SHADINGMODEL_CLEAR_COAT || MATERIAL_SHADINGMODEL_TWOSIDED_FOLIAGE || MATERIAL_SHADINGMODEL_HAIR || MATERIAL_SHADINGMODEL_CLOTH || MATERIAL_SHADINGMODEL_EYE) || MATERIAL_SHADINGMODEL_QTOON)
// TODO: fix end
```



### 6. DeferredShadingCommon.ush

找到 `IsSubsurfaceModel` 和 `HasCustomGBufferData` 

添加我们的判断

表示如果 `ShadingModel == SHADINGMODELID_QTOON` ，`IsSubsurfaceModel` 就为 true

如果 `ShadingModelID == SHADINGMODELID_QTOON`, `HasCustomGBufferData` 就为 true

```c++
bool IsSubsurfaceModel(int ShadingModel)
{
	// TODO: fix begin, QToon // shader 6
	return ShadingModel == SHADINGMODELID_SUBSURFACE 
		|| ShadingModel == SHADINGMODELID_PREINTEGRATED_SKIN 
		|| ShadingModel == SHADINGMODELID_SUBSURFACE_PROFILE
		|| ShadingModel == SHADINGMODELID_TWOSIDED_FOLIAGE
		|| ShadingModel == SHADINGMODELID_HAIR
		|| ShadingModel == SHADINGMODELID_EYE
		|| ShadingModel == SHADINGMODELID_QTOON;
	// TODO: fix end
}

bool UseSubsurfaceProfile(int ShadingModel)
{
	return ShadingModel == SHADINGMODELID_SUBSURFACE_PROFILE || ShadingModel == SHADINGMODELID_EYE;
}

bool HasCustomGBufferData(int ShadingModelID)
{
	// TODO: fix begin, QToon // shader 6
	return ShadingModelID == SHADINGMODELID_SUBSURFACE
		|| ShadingModelID == SHADINGMODELID_PREINTEGRATED_SKIN
		|| ShadingModelID == SHADINGMODELID_CLEAR_COAT
		|| ShadingModelID == SHADINGMODELID_SUBSURFACE_PROFILE
		|| ShadingModelID == SHADINGMODELID_TWOSIDED_FOLIAGE
		|| ShadingModelID == SHADINGMODELID_HAIR
		|| ShadingModelID == SHADINGMODELID_CLOTH
		|| ShadingModelID == SHADINGMODELID_EYE
		|| ShadingModelID == SHADINGMODELID_QTOON;
	// TODO: fix end
}
```



### 7. ShadingModels.ush <font color=red>核心</font>

参考代码 [UE4 Custom Shading Model - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/212785666)

#### 7.1 ToonStep 函数

```c++
float3 ToonStep(float feather, float halfLambert, float threshold = 0.5f)
{
    return smoothstep(threshold - feather, threshold + feather, halfLambert);
}
```

实现 Toon 渲染的 **色阶** 效果（即卡通渲染中的轮廓效果）。它通过 `smoothstep` 函数，根据输入的 `halfLambert`  **光照值** 和 **阈值**  `threshold` 及 **羽化量**  `feather` 来生成平滑的色阶变化，从而形成 Toon 渲染中的轮廓边缘。



##### 7.1.1 `smoothstep` 函数

- 渲染中常用的插值函数，能生成 **平滑的过渡效果**。
- 基本作用
    - **将输入值从一个范围映射到另一个范围，并在映射过程中生成一个光滑的插值**



- **函数原型**

  ```c++
  float smoothstep(float edge0, float edge1, float x);
  ```

- **参数说明**

  - `edge0`
    - 下限值。 `if (x < edge0) return 0` 注意这里是输出 0 不是输出 `edge0`
  - `edge1`
    - 上限值。`if (x > edge1) return 1` 注意这里是输出 1 而不是输出 `edge1`
  - `x`
    - 输入值，通常在 `[edge0, edge1]` 之间

- **工作原理**

  - `smoothstep` 使用 <font color=red>**三次插值函数 cubic interpolation**</font> 在 `edge0` 和 `edge1` 之间生成一个光滑的曲线。

  - **数学表达式**

    - `edge0 < x < edge1` 时，`smoothstep` 函数可以表示为
      $$
      \bold {smoothstep}(edge0, edge1, x) = t \times t \times (3 - 2 \times t)
      $$

      > chatgpt 给出的这个公式不大好记，我觉得记成这样比较方便
      >
      > $\bold {smoothstep} = 3t^2 - 2t^3$

      其中，`t` 是输入值 `x` 在 `edge0` 和 `edge1` 之间的归一化值，计算方法为
      $$
      t = \frac{x - edge0}{edge1 - edge0}
      $$

      > 至于 t ，实际上就是按照比例找到 x 在 所限制值域范围（`[edge0, edge1]`）内的对应值

- **用途**

  - `smoothstep` 常用于图形编程和 shader 中，用来创建 **光滑的渐变、淡入淡出效果、模糊边缘** 以及类似的视觉效果。相比于 **线性插值**，`smoothstep` 在起点和终点的过渡更加自然，不会出现突然的变化。

- **示例**

  - 例如，调用 `smoothstep(0.2, 0.8, 0.5)` 时，

    x = 0.5, edge0 = 0.2, edge1 = 0.8,
    $$
    t = \frac{0.5 - 0.2}{0.8 - 0.2}= 0.5
    $$

    $$
    result = t^2 \times (3 - 2 \times t) = 0.25 * 2 = 0.5
    $$




##### 7.1.2 threshold 和 feather 的说明

根据链接 [图形算法与实战：5.图像边缘羽化专题（1）滤波方法羽化_羽化算法-CSDN博客](https://blog.csdn.net/qq_32391345/article/details/106886043)

- 图像羽化

    - 是指图像边缘以渐变的方式，以达到逐渐朦胧或者虚化的效果

- 图像处理前

    <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408291530366.png" alt="image-20240829153012311" style="zoom:50%;" />

- 图像处理后

    <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408291530753.png" alt="image-20240829153037687" style="zoom:50%;" />



- 为什么使用 `threshold - feather` 和 `threshold + feather` 作为 edge0 和 edge1?
    - 控制过渡区域的 **范围和光滑度**。这种做法在图形渲染中非常常见，特别是在需要平滑过渡的情况下，比如实现 Toon 渲染中的边缘羽化效果。
    - **阈值 threshold**
        - 决定 **<font color=red>过渡中心</font>** 的位置，即在哪个 `x` 值附近会发生主要的过渡。例如，在 Toon 渲染中，<font color=red>`threshold` 可以设为高光或阴影的分界点</font>.
    - **羽化量 feather**
        - 控制 <font color=red>**过渡区域的宽度**</font>。
        - 通过增加或减少 `feather`，可以控制过渡区域是更宽还是更窄，即过渡是更平滑还是更突然。
- 应用场景
    - 在 Toon 渲染中，这种处理可以用来平滑地过渡阴影或者高光区域，从而避免过于突兀的视觉效果。例如：
    - **阴影边缘羽化**：如果没有羽化处理，阴影与亮部的分界线可能会显得过于生硬。通过使用 `threshold - feather` 和 `threshold + feather`，可以让这条分界线更柔和，给人一种更自然的渐变效果。
    - **高光处理**：类似地，高光的边缘也可以通过这种方式进行柔化处理，使得光照在物体表面上有一种更自然的扩散效果。





































