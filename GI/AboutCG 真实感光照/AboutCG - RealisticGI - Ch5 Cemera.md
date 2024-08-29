# AboutCG - RealisticGI - Ch5 Cemera

- 空间变换
    - 变换矩阵 SRT
      - scale, rotate, translate 缩放，旋转，平移
    - 矩阵乘法
- 相机空间
    - World space -> Camera Space 相机矩阵
    - Camera space -> World Space
- 透视投影
    - 近大远小
    - 投影矩阵





## 10. 图形流水线

GPU 是怎么绘制 3D 几何体的



















## 13. 射线数学原理

- 屏幕像素发出射线
  - 算法流程





## 14. 接口设计

```glsl
// RaytracingTutorial
// Author: Icewalnut

Ray pixel_to_ray(
	in const float2 pixel, 					// [0...width), [0...height)
    in const uint2 resolution,				// 分辨率
    // 因为算出来的点和相机的连线，这个射线是在相机的局部坐标的，要把这个射线转换到世界坐标 
    in const float3 ws_camera_pos,			// 相机坐标（在世界空间中的位置）
    in const Mat4f camera_to_world,			// view space 也就是相机的局部空间 到 world space 的变换矩阵
    in const Mat4f project_matrix			// 投影矩阵（view space 到 clip space）
)
```



相机空间 -> 投影空间 -> NDC 空间 -> UV 空间 -> VP 空间

现在 视口空间就是参数中的  pixel ，转回去的话要逆着来。先转换到 UV 空间，就是 [0, 1] 的范围，再转到 NDC 空间，再转到 投影空间，最后再转到相机的局部空间，也就是 view space

即 VP -> UV -> NDC -> Proj -> View Space

因此这里需要 project_matrix 做这一步逆向的转换



## 15. 透视转射线

- Camera.fx

  ```glsl
  Ray pixel_to_ray(
  	in
  )
  ```

  









































