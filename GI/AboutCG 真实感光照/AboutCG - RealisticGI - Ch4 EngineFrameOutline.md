# AboutCG - RealisticGI - Ch4 EngineFrameOutline

[ABOUTCG|高端CG教程-数字教育_ABOUTCG视频教程](https://www.aboutcg.org/play?courseId=1217&lessonId=76849)



- **基础渲染框架**
    - 框架流程，包括初始化、引擎的每一帧怎么循环、引擎的模块怎么更新，图形模块怎么刷新一帧，怎么把一帧的渲染结果输出到屏幕
    - camera 相机，Viewpot 视口
    - 用相机渲染。本章只需要知道怎么使用接口，后面的章节会讲解具体实现原理
    - 创建一个最简单的 shader



- Demo.hpp

```c++
#pragma once
```







- main.cpp

```c++
#include "raytracing/Demo.hpp"
#include <gui/GuiSystem.hpp>
#include <middleware/camera/CameraControllerView.hpp>
#include <../app/render/Shader.hpp>

auto demo_entry = ___PathTracerSimple;

int main()
{
    using namespace tutorial;
    gfx::setup();
    
    int window_width = 1024;
    int window_height = 768;
    auto window = hw::Window{
        hw::VideoMode(window_wid)
    }
}
```





## 8. 光线追踪入口模块

**ScreenRayTracer**

和普通的 shader 相比，会多出一个生成光线的部分

这个 shader 会包括两个部分，一个是 vertex shader，一个是 着色部分的 shader

- **filter** 用于绘制全屏的 **quad** 顶点 **shader**
- 光线追踪的着色部分的 shader
  - 着色部分一部分用来生成射线
  - 一部分用来着色
    - 这样做的好处是，以后可以直接把着色的部分替换掉，因为生成射线这部分的代码是可以重复使用的，可以抽取出来，会变的地方主要是着色的部分，着色的部分会经常的改动。所以一开始就要做这样的分离操作，这样更方便维护



![image-20240614001551520](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406140015538.png)

<p>图1</p>

这里看下怎么用矩形挡住摄像机。因为相机一直在旋转，一直在走，实现最方便的做法就是用它的局部空间

它的局部空间也是由 3 个轴构成的空间，相机的视点就在 (0, 0, 0) 点。相机有 near clip plane 和 far clip plane ，近裁剪平面就是要投影图像的地方。

![image-20240614002548048](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406140025072.png)

<p>图2</p>

**near clip plane 用的是 NDC 空间**

![image-20240614002817411](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406140028434.png)

<p>图3</p>

NDC 空间就是 归一化设备坐标，x 和 y 都是从 -1 到 1 的。（好像 DX 中是 从 0 到 1 的？？？）

如果要用一个矩形来挡住这个摄像机，就是在  NDC 坐标空间中，画一个矩形，4个顶点，会分隔成两个 三角形。也就是说，在这个 NDC 里面，画两个三角形就能把相机挡住了。

![image-20240614003151736](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406140031753.png)

<p>图4</p>

这就是整个顶点 shader 的流程



引擎会输入 4 个顶点，将这 4 个顶点放到上图中的 4 个角，它就会生成两个三角形，这两个三角形经过扫描线，光栅化之后，就会得到图 1 中的那些像素

![image-20240614003715993](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406140037011.png)

然后我们在 pixel shader 中进行着色

这部分的代码引擎已经直接提供好了，所以我们的重点就是关注如何产生射线，如何着色

投影的部分就不用过多关注了





## 9. 全屏 shader

### 9.1 顶点 shader

#### 9.1.1 Shader

四边形投影的 shader 已经在引擎中实现了

![image-20240614082718432](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406140827461.png)

`PostProcessVs.fx`

```c++
#ifdef VERTEX_SHADER

static const float4 g_pos[] = 
{
    float4(-1, -1, 0, 1),
    float4(-1, 1, 0, 1),
    float4(1, -1, 0, 1),
    float4(1, 1, 0, 1),
};

// 纹理坐标注意左上角是 (0, 0)，右下角是 (1, 1)
static const float2 g_texcoord[] = 
{
	float2(0, 1),
    float2(0, 0),
    float2(1, 1),
    float2(1, 0),
};

/**
 * 顶点 shader 的入口函数
 * vertexID: 顶点 id。我们会在 c++ 中输入 4 个顶点，它们的 id 分别是 0,1,2,3, 
 */
void main(uint vertexID : SV_VertexID,
         out float4 ndc_pos : SV_Position,
         out float2 texcoord : TEXTURE0)
{
    // 用这些顶点的 id 来找到它们的坐标
    ndc_pos = g_pos[vertexID];
    // 纹理坐标
    texcoord = g_texcoord[vertexID];
}

#endif
```

![image-20240614084323113](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406140843130.png)

纹理坐标注意左上角是 (0, 0)，右下角是 (1, 1) 。和 OpenGL 中不同



#### 9.1.2 HLSL中的 semantic 语义

```glsl
void main(uint vertexID : SV_VertexID, out float4 ndc_pos : SV_Position, out float2 texcoord : TEXTURE0)
```

其中 `SV_VertexID`, `SV_Position` 和 `TEXTURE0` 是 **语义**

1. **SV_VertexID**

    **SV** 是 System Value 的缩写

    是一个 **系统值** 语义 semantic

    它在每个顶点 shader 调用时提供一个唯一标识符，表示当前处理的事顶点 buffer 中的哪一个顶点。

    这对于基于 **索引** 的数据访问（例如本例中访问顶点位置和纹理坐标数组）非常有用

2. **SV_Position**

    是一个系统值语义 semantic

    代表 VS 输出的位置

    这个值会被后续的阶段（例如光栅化阶段）使用，以确定最终像素在屏幕上的位置。他是一个 四维向量（`float4`），其中前三个分量表示归一化设备坐标 NDC，最后一个分量通常用于表示 <font color=red>深度信息</font>

3. **TEXTURE0**

    这是一个自定义的 semantic，用于指定 **输出到纹理坐标寄存器**。

    在本例中，`texcoord` 将会被用作纹理采样的坐标。虽然这个名称看起来像输入纹理，但实际上它在这里作为输出语义，表明这些纹理坐标将被传递给后续的截断（如 PixelShader ）使用。







### 9.2 引擎中的实现

`render/Filter.hpp`

```c++
#pragma once 
#include <render/Shader.hpp>
#include <render/Viewport.hpp>
#include <gfx/Gfx.hpp>

namespace turotial::graphics
{
    class Filter final
    {
        public:
        static auto draw(ShaderVersion shader) -> void;
        static auto draw(const gfx::RTV& dst, const gfx::Viewport& rect, ShaderVersion shader) -> void;
        static auto draw(const Viewport& dst, ShaderVersion shader) -> void;
    };
}
```



`render/Filter.cpp`

```c++
#include <render/Filter.hpp>

namespace tutorial::graphics
{
    auto Filter::draw(ShaderVersion shader) -> void
    {
        gfx::bind_shader(shacer.fetch());
        gfx::set_primitive_type(gfx::PrimType::PRMI_TRISTRIP);
        gfx::set_blend_mode(gfx::BlendMode::Replace);
        gfx::set_depth_test(false, false, gfx::DepthFunc::DSS_DEPTHFUNC_ALWAYS);
        gfx::set_rasterize_mode(gfx::PolyCullMode::PCM_NONE, gfx::PolyFillMode::PFM_SOLID, true);
        gfx::bind_vertex_buffer(nullptr, nullptr);
        gfx::bind_index_buffer(nullptr);
        gfx::draw(4, 0);
    }
    
	/**
	 * const gfx::RTV& dst 渲染的目标的缓存
	 * const gfx::Viewport& rect 渲染的目标的缓存
	 * ShaderVersion shader 用这个 shader 来渲染全屏的矩形
	 */
	auto Filter::draw(const gfx::RTV& dst, const gfx::Viewport& rect, ShaderVersion shader) -> void
	{
		// 绑定帧缓存
		gfx::bind_framebuffer(gfx::empty_v<gfx::DSV>, { dst });
		// 设置视口
		gfx::set_viewport(rect);
		// 绘制 shader
		draw(shader);
		// 解绑 帧缓存
		gfx::unbind_framebuffer(gfx::empty_v<gfx::DSV>, { dst });
	}
    
    auto Filter::draw(const Viewport& dst, ShaderVersion shader) -> void
    {
        draw(dst.target, dst.rect, shader);
    }
}
```



## 10. 光线追踪入口模块

- ScreenRayTracer.hpp

```c++
#pragma once
#include <math/Math3d.hpp>
#include <render/Filter.hpp>
#include <render/Viewport.hpp>

namespace tutorial::graphics
{
	// 定义一个 ScreenRayTracer 的类，这就是光线追踪入口模块
	class ScreenRayTracer
	{
	private:
        // 构造函数被声明为 private 且被删除，表明禁止默认构造
		ScreenRayTracer() = delete;

	public:
		/**
		 * Viewport& viewport - 视口
		 * ShaderVersion shader - 用这个 shader 渲染全屏的矩形
		 * 后面的光线追踪的 shader 就是用这个接口来绘制全屏的渲染
		 */
		static void draw(Viewport& viewport, ShaderVersion shader);
	};
}
```

`ScreenRayTracer` 的构造函数被声明为 private 且被删除 (`= delete;`)，这样的做法有以下几个目的：

1. **禁止默认构造**

    这表明不希望用户直接通过默认构造函数创建 `ScreenRayTracer` 类的实例。

    当一个类没有显式定义任何构造函数时，编译器会自动为其生成一个默认 constructor，通过显式定义一个 private 的默认构造函数并将其删除，可以阻止这一行为，从而控制类的实例化方式。

2. **设计为单例或其它特殊初始化方式**

    这种设计通常暗示 `ScreenRayTracer` 类可能需要通过静态方法或工厂方法来获取其 **唯一实例**，或者需要在创建对象前执行特定的初始化逻辑。

    这是一种常见的设计模式，比如单例模式（Singleton Pattern），其中确保了类只有一个实例，并提供一个全局访问点。

3. **强调非实例化用途**

    有时一个类被设计为仅供继承使用（基类）或者只包含静态成员函数和静态数据成员，不需要创建实例。这种情况下，删除 constructor 可以清楚地传达这个意图给其他开发者。

在本例中，鉴于 方法 `draw` 是静态的，暗示 `ScreenRayTracer` 是为了渲染相关的功能设计的，这些操作不依赖于此类的实例状态，因此实例化该类可能并不符合其设计意图。





- ScreenRayTracer.cpp

```c++
#include "ScreenRayTracer2.hpp"
#include <core/Random.hpp>

namespace tutorial::graphics
{
	// 我们只需要创建一个 ScreenRayTracer 实例，因此创建一个 static 实例
	// 这里用了一些 RAII 的操作，也就是第一次用到的时候才会去初始化
	// 定义一个辅助的结构体
	struct SetupScreenRayTracer
	{
		SetupScreenRayTracer() noexcept
		{
			// 这里 _init 设置为 static ，是为了在不是 draw 函数调用 SetupScreenRayTracer 构造函数的时候
			// 拿到的是全局唯一的实例
			static bool _init = false;
			// 如果已经初始化了，就直接返回
			if (_init) return;

			// 如果没有初始化，就设置为 true
			// 这里所有的数据都是静态的，当前只有 _init 变量，还会用到什么数据后面再说
			// 因为是 static 的，只有在第一次进入的时候会初始化
			_init = true;
		}
	};

	// 用这个接口来绘制全屏的矩形
	void ScreenRayTracer::draw(Viewport& viewport, ShaderVersion shader)
	{
		// 上面定义的结构体，在第一次进入到这里的时候会初始化这个实例。
		// 初始化这个 __done 的时候就会调用这个 struct 中的构造函数。在构造函数中会判断是不是已经初始化了（_init）
		static SetupScreenRayTracer __done;
		Filter::draw(viewport.target, viewport.rect, shader);
	}
}
```

`draw` 函数中的变量 `__done` 是 static 的，也就是被声明了一次，在第一次进入 `SetupScreenRayTracer` 构造函数的时候，`_init` 是 false 的，然后被设置为 true。 在后面第二次进入 `draw` 函数的时候，`__done` 是静态的，且已经被初始化过了，就不会再进行第二次初始化了。

然后， `_init` 被设置为 static ，是为了在别的不是 `draw` 函数调用 `SetupScreenRayTracer` 构造函数的时候，确保拿到的 `SetupScreenRayTracer` 实例仍然是全局的唯一的实例。



以上就是光线追踪 shader 的入口框架



































