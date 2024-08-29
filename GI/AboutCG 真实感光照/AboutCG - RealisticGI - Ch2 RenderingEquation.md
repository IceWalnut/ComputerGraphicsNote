# AboutCG 真实感光照 - Ch2 渲染方程

- 全局光照渲染方程
  - $L_o$ 辐射亮度 radiance
  - 能量守恒
  - 光线追踪



### The Rendering Equation

**An object can emit light itself. It also receives light from different directions, which it will either reflect or absorb.**

![image-20240611232951011](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202406112329054.png)

Light exiting the surface  = Emitted light + reflected incoming light


$$
L_o(x, \vec {\omega_o}) = \underbrace{L_e(x, \vec\omega_o)}_{emitted} + \underbrace{\displaystyle \int_\Omega L_i(x, \vec\omega_i)f_r(\vec\omega, x, \vec\omega_i) \ \cos\theta \ \mathrm d\vec\omega_i}_{reflected \ incoming \ light}
$$


- $L_o(x, \vec\omega_o)$
  - exiting radiance in point $x$ towards direction $\vec\omega_o$
- $L_e(x, \vec{\omega_o})$
  - emitted radiance in point $x$ towards direction $\vec \omega_o$
- $\displaystyle \int _{\Omega} ... d\vec\omega_i$
  - sum of the incoming radiance from all $\vec{\omega_i}$ directions
- $L_i(x, \vec{\omega_i})$
  - incoming radiance from direction $\vec{\omega_i}$ to point $x$
- $f_r(\vec\omega_o, x, \vec\omega_i)$
  - BRDF
- $\cos \theta$
  - light attenuation
