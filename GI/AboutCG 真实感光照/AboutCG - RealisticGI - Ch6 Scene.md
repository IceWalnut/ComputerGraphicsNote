# AboutCG - RealisticGI - Ch6 Scene

- 定义几何体
  - 球，四边形
  - mesh bvh
- 定义材质
- 射线，场景相交测试
- 创建场景
- 输出材质

![image-20240703082257432](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407030822505.png)





## 1. 介绍

### 1.1 球和四边形的定义

- 球

  - 球心，半径 r

    ![image-20240703083325361](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407030833383.png)

- 四边形

  - 起点 P，向量 u, v

    面积 $A = |\vec{u}| \cdot |\vec{v}|$

    ![image-20240704082437739](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407040824771.png)

    法线 $\vec{n} = \displaystyle \frac{\vec u \times \vec v}{|\vec u||\vec v|}$ 代表四边形的朝向

    ![image-20240704082912875](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407040829897.png)

    如果眼睛在四边形的后面，是看不到这个四边形的。相交测试的时候要注意这一点。如果一根射线打到它的背后，是认为它不相交的。
    
    ![image-20240706133957761](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407061339788.png)



### 1.2 如何存储几何体

程序初始化的时候会初始化这个场景，告诉路径追踪渲染器这个场景什么样子。现在需要定义一个数据结构，即这些几何体的存储方式。

现在场景非常简单，只有 四边形和球形，只需要两个数组来存储。还需要两个数组，来存储灯光和材质。共 4 个数组。

![image-20240704083807971](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407040838995.png)

在 下一节讲解数据结构。



## 2. Entity

- Entity.fx

```c++
struct Entity
{
    int type;		// 类型，根据这个类型知道在哪个数组中寻找几何体
    int index;		// 上面数组中的索引
};

struct SphereMesh
{
    float3 position;
    float radius;
    int material;		// 这里 material 只是一个索引，用这个索引去材质数组中找具体的材质
};
    
struct QuadMesh
{
    float3 position;
    float3 u;
    float3 v;
    int material;
};
    
// Shader 中没有构造函数，需要自己创建实例，也就是自己实现构造函数
SphereMesh ___SphereMesh(in float3 position, float radius, int material)		// 这里的 in 是可以省略的
{
    SphereMesh ball = (SphereMesh)0;		// 将 SphereMesh 所有成员变量初始化为 0
    ball.position = position;
    ball.radius = radius;
    ball.material = material;
    return ball;
}

QuadMesh ___QuadMesh(in float3 position, float3 u, float3 v, int material)
{
    QuadMesh quad = (QuadMesh)0;
    quad.position = position;
    quad.u = u;
    quad.v = v;
    quad.material = material;
    return quad;
}
```



## 3. Light

### 3.1 定义光源

- light.fx

```c++
static const int QuadLight = 0;			// 四边形的面积光
static const int SphereLight = 1;		// 点光源

struct Light
{
	float3 position;					// 光源的位置（四边形光源是起点）
	float radius;						// 光源的半径   点光源会用到
	float3 energy;						// 辐射亮度
	float type; // 0 : quad, 1 : sphere 光源类型
	float3 u;							// 四边形光源用到 u 和 v
	float area;
	float3 v;
};

Light ___QuadLight(
	in float3 position, 
	in float3 energy,
	in float3 u,
	in float3 v
) 
{
	Light light = (Light)0;
	light.position = position;
	light.energy = energy;
	light.u = u;
	light.v = v;
	light.type = QuadLight;
	return light;
}
```



### 3.2 场景的数据结构

- Scene.fx

```c++

struct Scene
{
    uint nball;		// 场景中球形的个数
    uint nquad;		// 场景中四边形的个数
    uint nlight;	// 场景中光源的个数
    uint pad;		// 最后加一个 32 位整型的 pad 只是为了 16 bytes 的对齐
};
    
// 整个场景中只有一个球
#define SphereMeshBuffer(xxx) SphereMesh xxx[1]		
#define QuadMeshBuffer(xxx) QuadMesh xxx[6]
#define LightBuffer(xxx) Light xxx[1]
```

这里用宏定义来实现几个几何体的数组定义。

`#define SphereMeshBuffer(xxx) SphereMesh xxx[1]` 就是一个长度为 1 的球形网格数组。因为写一个结构体太麻烦了，所以用宏定义。

> 注意 `#define` 这一行的最后是没有分号的

上面代替了

```c++
struct SphereMeshBuffer
{
    ShereMesh buffer[1];
};
```

宏定义就是字面上代替整个代码，将代码的文本直接替换掉。



## 4. 材质

这节课用的材质非常简单，材质数据只有一个 basecolor 也就是颜色。用一个 `float3` 来表示这个颜色。

下面的代码在 `PathTracerSimple.fx` 中定义 Material 这个结构体

```c++
struct Material { float3 baseColor; };
    // 构造函数
Material ___Material(float3 baseColor)		// 输入一个 baseColor 
{
	Material material;
    material.baseColor = baseColor;			// baseColor 给这个材质，然后返回
    return material;
}

#define MaterialBuffer(xxx) Material xxx[4]
```

上面的材质数组有 4 个元素，也就是现在只用到了 4 个材质。



## 5. 表面信息

接下来来实现相交测试。

当一根 ray 打中一个物体的时候，我们要记录这个点的信息。包括这个点的表面信息，叫 surface ，还有相交的信息，叫 hit.

![image-20240705083641781](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407050836803.png)



### 5.1 表面信息

- 表面

  - 要记录点的三维坐标 (x, y, z)

  - 这一点的材质 material

  - 这一点的法线 normal N

    - 需要注意的是，不仅要记录 <font color=red> 几何体 geometry 的法线 （所谓几何体的法线是指物体本身的法线 N）</font>

    - 还要记录它的 <font color=red>**正面法线 Front Face Normal, FFNormal**</font>，下面是解释

      ![image-20240705082943859](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407050829884.png)

      光线射入物体内部，反射或折射到物体内部表面时，它的法线是一个虚拟的法线，我们又叫作 **正面法线 front face normal, FFNormal**

接下来定义这一点的表面信息（相交信息）

- Scene.fx 中

```c++
struct Surface
{
    float3 position;
    int material;		// material id
    float3 normal;
    float3 ffnormal;	// 正面法线
};
```



### 5.2 相交信息

![](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407050836603.png)

当一根 ray 射向一个物体的时候，需要知道这根 ray 的起点，及它到物体表面交点的距离，以及打中的这个实例 entity，还有打中的这一点的表面信息。

现在来定义这个结构体

- Scene.fx 中

```c++
struct Hit
{
    Entity entity;		// 打中的 entity     Entity 看上面，只有一个 type 和 index
    float hit_dist;		// 打中的距离
    bool hit;			// 是否打中
    Surface surface;	// 表面信息
};
```

实现一个构造函数，Hit 的初始化是什么也没有打中

```c++
Hit NotHit()
{
    static Surface surface = (Surface)0;
    // Entity 用默认的，距离用无限远，是否打中用 false
    static Hit not_hit = { { 0, -1 }, INFINITY, false, surface };
    return not_hit;
}
```

注意这里的  `INFINITY` 会在 `defines.fx` 中进行定义

```c++
static const float INFINITY = 1e9;
```



## 6. 射线求交

- Geometry.fx

### 6.1 与球相交

```c++
// 如果没有定义 GEOMETRY，则定义它
#ifndef GEOMETRY
#define GEOMETRY
// static const float INFINITY = 1e9;
// static const float EPS = 0.01f;
// static const float PI = 3.1415926535897323f;
// static const float TWO_PI = 6.28318530717958648f;

struct Box { float3 min; float3 max; };
struct Ray { float3 origin; float3 direction; };
struct Sphere { float3 position; float radius; };

// return hit distance, or INFINITY if not intersect
float ray_sphere_intersect(float3 pos, float radius, in Ray r)
{
    float3 op = pos - r.origin;				// 射线起点到 位置 Pos 的向量
    float eps = 0.001;						// epsilon, 小值，用于浮点数比较
    float b = dot(op, r.direction);			// op 和 射线的点积
    float det = b * b + radius * radius - dot(op, op);		// 判别式，判断射线是否与球体相交
    [branch]								// 这是一个提示编译器的分支预测指令，表示之后的 `if` 语句是一个分支
    if (det < 0.0)							// 射线没有与球体相交，返回 INFINITY
        return INFINITY;
    
    det = sqrt(det);
    float t1 = b - det;
    [branch]
    if (t1 > eps)
        return t1;
    
    float t2 = b + det;
    [branch]
    if (t2 > eps)
        return t2;
    
    return INFINITY;
}
// 与矩形相交见下面
...
#endif // GEOMETRY
```

下图解释了上面是否相交的算法

![image-20240706112110313](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407061121383.png)

- 相切
    - 从 P 发出的 ray 与圆相切于点 A，则 $PA^2 + r^2 = OP^2$
- 相离
    - OP 在 从 P 发出的 ray 方向上的投影长度为 $PB$，即 $dot(op, r.direction) = PB = b$
    - $PBA$ 组成一个钝角三角形，故 $PA > PB$
    - 故 $det = b^2 + r^2 - OP^2 < 0$

- 相交
    - OP 在 从 P 发出的 ray 方向上的投影长度为 $PE$，即 $dot(op, r.direction) = PE = b$
    - $PAE$ 组成一个钝角三角形，$PE > PA$
    - 故 $det = b^2 + r^2 - OP^2 > 0$

- 求解距离 PC 或者 PD
    - PC
        - $PC = PE - CE = b - CE = b - \sqrt{r^2-OE^2} = b - \sqrt{r^2-(OP^2 - PE^2)} =   b - \sqrt{r^2 + b^2 - OP^2} = b- \sqrt{det} $
    - 同理，$PD = b + \sqrt{det}$



### 6.2 与矩形相交

`Geometry.fx` 中另一个函数 `ray_rect_intersect`

```c++
// return hit distance, or INFINITY if not intersect
float ray_rect_intersect(in float3 pos, in float3 u, in float3 v, in float3 n, in Ray r)
{
    float dt = dot(r.direction, n);				// 射线方向和矩形法线 n 的点积，因为都是单位向量，这个点积就是 cos\theta
    float w = dot(n, pos);						// 法线和矩形位置 pos 的点积 w
    float t = (w - dot(n, r.origin)) / dt;		// 计算相交点的距离 t  距离的计算原理见下图
    
    [branch]
    if (t > EPS)
    {
        float3 a = r.origin + r.direction * t;	// 射线与四边形交点 A 的位置
        float3 pa = a - pos;					// 向量 PA
    
    	// pa project onto u, v, and its length <= |u| and |v|
    	// 0 <= dot(normalize(u), pa) <= length(u)
        // ---->  0 <= dot(normalize(u)/length(u), pa) <= 1
        // ---->  0 <= dot(u/dot(u,u), pa) <= 1
        u = u / dot(u, u);						// 将 u,v 归一化
        v = v / dot(v, v);						
        float u1 = dot(u, pa);					// PA 在 u 方向上的投影
        
        [branch]
        if (u1 >=0 && u1 <= 1)					// 判断点 A 是否在四边形内部
        {
            float v1 = dot(v, pa);
            [branch]
            if (v1 >=0 && v1 <= 1)
            {
                return t;
            }
        }
    }
    
    return INFINITY;
}
#endif // GEOMETRY
```



![image-20240706133915475](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407061339511.png)

如上图，假设射线 $\vec r$ 与四边形相较于点 $A$，$\vec r$ 的起点 $R$ 在四边形同平面的映射为 $R'$，则距离 $RA = RR'/\cos\alpha$

而 $\cos\alpha = -\cos\theta = -dot(r.direction, n)$ (因为 $r.direction$ 与 $n$ 都是单位向量)

又 $RR' = R 点在 \vec n 方向的映射 - P点在 \vec n 方向的映射 = dot(n, r.origin) - dot(n, pos)$

故距离 $t = ((dot(n, pos) - dot(n, r.origin)) / dot(n, r.direction)$

如果 $t < 0$ 说明射线和 法线 n 同向，说明和四边形没有相交



## 7. 光源求交

在 scene.fx 中，定义一个 `trace_light` 函数返回射线到光源的距离，输入的是一个射线和一个光源

> 注意，这里的 ray 可能并不是光，可能是从摄像机发出的一条射线，因此要寻找和这个射线相交的光源/球体/四边形中最近的那一个 entity

<font color=red>**相交测试就是返回距离射线最近的 entity**</font>

```c++
float trace_light(in const Ray ray, in const Light light)
{
    [branch]
    switch (light.type)
    {
            // 先实现四边形面积光
        case QuadLight:
            {
                float3 u = light.u;
                float3 v = light.v;
                float3 normal = normalize(cross(u, v));		// 用 u 和 v 算出法线的朝向
                
                [branch]
                if (dot(normal, ray.direction) >= 0)		// 如果射线和法线不同向才说明能打中面
                    return INFINITY;
                return ray_rect_intersect(light.position, u, v, normal, ray);		// 调用射线和四边形的相交测试
            }
         default:
            break;
    }
    return INFINITY;
}
```

上面这个函数做了射线和单个灯光的相交测试。

后面遍历整个场景中所有的光源，逐个做相交测试，并且返回和这个射线最近的光源

```c++
void trace_scene_lights(
	in const Ray ray, 				// 输入射线
    in Scene scene,					// 整个场景
    in LightBuffer(lights),			// 光源
    inout Hit hit					// 相交信息 Hit，是既要读又要写的
)
{
    // 把最近的相交距离先存起来
    float t = hit.hit_dist;			// 后面只会记录比这个 t 要小的光源(所以初始的时候，输入的 hit 中的 hit_dist 可能是 INFINITY?)
    uint light_count = scene.nlight;
    
    [loop]
    for (uint i = 0; i < light_count; ++i)
    {
        Light light = lights[i];
        float d = trace_light(ray, light);		// 算出射线到光源的距离
        
        [branch]
        if (d < t)								// 只有射线到光源的距离 < hit_dist 才会记录
        {
            t = d;								// 同时保留当前的相交距离作为最近的相交距离
            hit.entity.index = i;
            hit.entity.type = LIGHT_ENTITY;
            hit.hit_dist = t;
            hit.hit = true;
        }
    }
}
```

接下来实现射线和场景中所有球形的相交测试

```c++
void trace_scene_sphere(in const Ray ray, in Scene scene, in SphereMeshBuffer(balls), inout Hit hit)
{
    float t = hit.hit_dist;
    
    [loop]
    for (uint i = 0; i < scene.nball; ++i)
    {
        SphereMesh sphereMesh = balls[i];
        // d 是 ray 和 sphere 相交的距离
        float d = ray_sphere_intersect(sphereMesh.position, sphereMesh.radius, ray);
        
        [branch]
        if (d < t)
        {
            t = d;
            hit.entity.index = i;
            hit.entity.type = SPHERE_ENTITY;
            // 表面信息 位置
            hit.surface.position = ray.origin + ray.direction * d;
            // 因为是球，所以法线就是 交点 - 球心 再归一化
            hit.surface.normal = normalize(hit.surface.position - sphereMesh.position);
            // 如果表面法线和射线方向夹角 > 90 度（如下图 C 点），说明交点是球外表面，则正面法线就是原法线
            // 如果表面法线和射线方向夹角 < 90 度（如下图 D 点），说明内表面，则正面法线是 -normal
            // 也就是用点积来判断是否同向
            hit.surface.ffnormal = dot(hit.surface.normal, ray.direction) <= 0.0 ? 
                hit.surface.normal : hit.surface.normal * -1.0;
            hit.surface.material = sphereMesh.material;
        }
    }
    
    [branch]
    if (t < hit.hit_dist)		// 最后判断 t 是不是小于最近相交距离，如果有就记录
    {
        hit.hit_dist = t;
        hit.hit = true;
    }
}
```

![image-20240706153034276](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407061530316.png)





## 8. Mask 掩码

后面要做的是 射线 怎么和场景中所有的 entity 做相交测试。在此之前，先要定义**掩码**。

所谓掩码就是，相交测试是射线跟 **哪一类物体** 做相交测试。

在 `Entity.fx` 中定义了 entity 的类型

```c++
// enum EntityType
static const int MESH_ENTITY = 1;
static const int LIGHT_ENTITY = 2;
static const int SPHERE_ENTITY = 5;
static const int QUAD_ENTITY = 6;
```

接下来实现掩码，掩码就是二进制

```c++
static const int MASK_MESH = 1 << MESH_ENTITY;			// 0000 0001 左移 1 位 即 0000 0010
static const int MASK_LIGHT = 1 << MESH_LIGHT;			// 0000 0100
static const int MASK_SPHERE = 1 << SPHERE_ENTITY;		// 0010 0000
static const int MASK_MESH = 1 << QUAD_ENTITY;			// 0100 0000

// MASK_ALL 就是所有位置都有 1
static const uint MASK_ALL = MASK_MESH | MASK_SPHERE | MASK_QUAD | MASK_LIGHT;		// 0110 0110
// MASK_GEOMETRY 就是所有几何体
static const uint MASK_GEOMETRY = MASK_MESH | MASK_SPHERE | MASK_QUAD;
```

专门做阴影测试的时候会用到 `MASK_GEOMETRY`

假如我们要测试一个点和一个光源之间有没有几何体，只需要射线和所有的几何体做相交测试。`MASK_GEOMETRY` 就是在此时用的。

![image-20240706163747406](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407061637433.png)



## 9. 场景求交

回到 `Scene.fx` 这个文件。在这里实现一条 ray 怎么和场景中所有 entity 做相交测试

- Scene.fx

```c++
void trace_scene(
		in const Ray ray,
    in const float max_dist,		// search distance 搜索的距离。这样做的好处在于包容性，即给定一个距离，判断这个距离内有没有物体
    in Scene scene,
    in SphereMeshBuffer(balls),		// 输入所有的球体数组
    in QuadMeshBuffer(quads),		// 所有的四边形数组
    in LightBuffer(lights),			// 所有的光源数组
    in const uint mask,
    out Hit hit
)
{
    hit = NotHit();					// 初始化 hit = 什么也没有打中
    hit.hit_dist = max_dist;		// 相交距离就是输入的最大距离
    
    // hit 记录了最短的相交距离
    // 下面调用不同物体的相交测试
    // 用掩码判断有没有这类物体
    [branch] if ((mask & MASK_SPHERE) > 0) { trace_scene_sphere(ray, scene, balls, hit); }
    [branch] if ((mask & MASK_QUAD) > 0) { trace_scene_quad(ray, scene, quads, hit); }
    [branch] if ((mask & MASK_LIGHT) > 0) { trace_scene_lights(ray, scene, lights, hit); }
    // 上面三条结束了就知道哪一个是最近相交的点
}
```



## 10. 搭建场景

在 `PathTracerSimple.fx` 中，首先尝试初始化一个场景，然后输出这个场景的材质。

场景中只有 6 面墙，一个球，还有一个四边形的光源。

### Random

首先，`Random` 结构体的定义在 `Random.fx` 中，

```c++
struct Random
{
    float2 seed;
    float2 random_vec;
};
```



### PathTracerSimple

```c++
float3 pathTracing(in Ray ray, inout Random random)
{
    // init scene
    Scene scene = (Scene)0;			// 看上面 Scene 结构体中实际上只有 ball, squd, light 的个数
    
    MaterialBuffer(materials);		// 材质数组，这里使用了宏替换 ，数组中有 4 个元素
    materials[0] = ___Material(float3(1, 1, 1));		// 输入一个 baseColor 白色
    materials[1] = ___Material(float3(1, 0, 0));		// 红色
    materials[2] = ___Material(float3(0, 0, 1));		// 蓝色  rgb, red , green , blue
    materials[3] = ___Material(float3(0.5, 0.5, 0.5));	// 灰色
    int white = 0;										// 给索引对应的名字
    int red = 1;
    int blue = 2;
    int grey = 3;
    
    SphereMeshBuffer(balls);
    balls[0] = ___SphereMesh(float3(0, -1.5, 0), 1, white);		// 参数： 位置， 半径， 材质
    balls[0].mateial = 0;										// 球的材质使用 0 号材质（不懂这里为什么又写一次, white 就是 0 啊）
    scene.nball = 1;
    
    QuadMeshBuffer(quads);
    quads[0] = ___QuadMesh(float3(-2.5f, 2.5f, 2.5f), float3(5, 0, 0), float3(0, -5, 0), grey);		// front 下图中 A 点
    quads[1] = ___QuadMesh(float3(-2.5f, 2.5f, -2.5f), float3(0, 0, 5), float3(0, -5, 0), red);		// left        E 点
    quads[2] = ___QuadMesh(float3(2.5f, 2.5f, 2.5f), float3(0, 0, -5), float3(0, -5, 0), blue);		// right	   B 点
    quads[3] = ___QuadMesh(float3(-2.5f, -2.5f, 2.5f), float3(5, 0, 0), float3(0, 0, -5), grey);	// down        D 点
    quads[4] = ___QuadMesh(float3(-2.5f, 2.5f, -2.5f), float3(5, 0, 0), float3(0, 0, 5), grey);		// up		   E 点
    quads[5] = ___QuadMesh(float3(2.5f, 2.5f, -2.5f), float3(-5, 0, 0), float3(0, -5, 0), grey);	// back        F 点
    // 上面 quads[5] 有疑问，我的想法和源代码中的不一样
    // 源码： quads[5] = ___QuadMesh(float3(2.5f, 2.5f, -2.5f), float3(-5, 0, 0), float3(0, 0, -5), grey);	// back
  
  	// lights
  	LightBuffer(lights);
  	// position, energy, u, v
  	lights[0] = ___QuadLisht(
    		float3(-1.f, 1.5f, -2.5f),		// position
      	float3(1, 1, 1) * 3.14,				// energy
      	float3(2, 0, 0),							// u
      	float3(0, 0, 2)								// v
    );
  	scene.nlight = 1;
}
```

![image-20240707101324308](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407071013347.png)

这样整个场景就搭建完毕了



## 11. 输出场景

现在来测试一下这个场景。已经知道了相机出发的射线，现在用这个射线和场景求交

- PathTracerSimple.fx

```c++
// 返回值为能量值
float3 pathtracing(in Ray ray, inout Random random)
{
  	...	
  
  	// first hit
    Hit hit;
    // 下面这个函数见第 9 节
    trace_scene(ray, INFINITY, scene, balls, quads, lights, MASK_ALL, hit);
    if (hit.hit)
    {
        if (hit.entity.type == LIGHT_ENTITY)// 如果摄像机射出的射线直接打中光源
        {
            int index = hit.entity.index;		// 索引
            return lights[index].energy;		// 返回光源的能量
        }
        else if (hit.entity.type == SPHERE_ENTITY || hit.entity.type == QUAD_ENTITY)
        {
          	Material material = materials[hit.surface.mateiral];
          	return material.baseColor;
        }
    } 
  	else
    {
      return float3(0, 0, 0);
    }
  	
  	return 0;
}
```







































