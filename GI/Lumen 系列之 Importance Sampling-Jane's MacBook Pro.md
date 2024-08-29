# Lumen 系列之 Importance Sampling

转载自知乎 [Lumen系列之Importance Sampling - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/713507304)





即使做了自适应的 Probe 和 Jitter，在当今显卡所能承担的光线数量下做出的结果还是有很多噪点

![img](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408090936877.webp)





## 多重重要性采样 MIS

Multi Importance Sampling

将不同的 pdf 结合在以期形成一个新的 pdf 来提高采样效率

![img](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202408090933275.webp)

橙色和绿色可能代表 BRDF 和 incoming radiance，蓝色是合成的新 BRDF



## Lumen 中 Importance Sampling 的优化

### BRDF PDF

在代码中的 `GenerateBRDF_PDF` 里用 `Compute Shader` 通过材质进行生成：

```c++
{
  // 根据摆放的 Probe 从 GBuffer 中获取材质信息
  const FLumenMaterialData Material = ReadMaterialData(PixelPos, PixelScreenUV);
  
}
```



