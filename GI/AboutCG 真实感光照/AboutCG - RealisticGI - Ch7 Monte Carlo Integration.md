# AboutCG - RealisticGI - Ch7 Monte Carlo Integration



## 1. 介绍

- 蒙特卡洛积分
    - 数值积分
        - 这里去看 \<Global Illumination> 中的第二章 "Sampling 怎么做" 
    - 高维函数 $f(x_0, x_1, x_2, ..., x_n)$
- 随机变量
- 期望
- 重要性采样



蒙特卡洛积分就是做随机数，只和随机数有关，所以是线性的，和函数的维度无关



## 2. 随机变量

A random variable $X$ is a value chosen by some random process. We will generally use capital letters to denote random variables, with exceptions made for a few Greek symbols that represent special random variables. Random variables are always drawn from some domain, which can be either discrete (e.g., a fixed set of possibilities) or continious (e.g., the real numbers $\mathbb{R}$). Applying a function $f$ to a random variable $X$ results in a new random variable $Y=f(X)$

- 样本空间

  - 如骰子，就是 1 2 3 4 5 6

- 随机变量总是落在样本空间内

  随机过程是有概率的。

  $x_i = 1, 2, 3, 4, 5, 6$

  这个概率是 $p(x_i)$

  在骰子游戏中，概率是均匀的，都是 $1/6$

  这个是离散型，样本空间是有限的



- 连续型样本空间，函数 $f(x)$，在值域内，假设随机变量 $Y=f(x)$。如果从中选一个点，这个概率就是 $p(Y_i)$，那么从 $f(x)$ 中随机选择一个点就叫做 **采样**，即采样这个函数。



## 3. 数学期望 Expectation

讲义摘自 [PBR-book 13.1 Background and Probability Review (pbr-book.org)](https://www.pbr-book.org/3ed-2018/Monte_Carlo_Integration/Background_and_Probability_Review)



从样本空间中挑出一个数 $f(x)$ 是采样 sample。在多次的随机过程中，得到的随机变量 x 或 y 的平均值会趋向于一个数，即数学期望。



The *expected value* $E_p[f(x)]$ of a function $f$ is defined as the average value of the function over some distribution of values $p(x)$ over its domain. In the next section, we will see how Monte Carlo integration computes the expected values of arbitrary integrals. The expected value over a **domain, $D$,** is defined as 
$$
E_p[f(x)] = \int _D f(x)p(x) \mathrm dx \tag{7.3.1}
$$
As an example, consider the problem of finding the expected value of the cosine function between 0 and $\pi$, where $p$ is uniform. <font color=red>Because the PDF $p(x)$ must integrate to 1 over the domain, $p(x) = 1/\pi$</font>, so 
$$
E[\cos x] = \int _0^\pi \frac{\cos x}{\pi} \mathrm dx = \frac{1}{\pi}(\sin \pi - \sin 0) =0 \tag{7.3.2}
$$
which is precisely the expected result. (Consider the graph of $cos x$ over $[0, \pi]$ to see why this is so.)

The *variance* of a function is the expected squared deviation of the function from its expected value. 

> - variance 方差

Variance is a fundamental concept for quantifying the error in a value estimated by a Monte Carlo algorithm. It provides a precise way to quantify this error and measure how improvements to Monte Carlo algorithms reduce the error in the final result. The variance of a function $f$ is defined as:
$$
V[f(x)] = E[(f(x) - E[f(x)])^2]  \tag{7.3.3}
$$


> - $\cos x$ 的积分是 $\sin x$
> - $\sin x$ 的积分是 $-\cos x$



![image-20240710075708924](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407100757991.png)
$$
E_a = 10 * 0.5 + 9 * 0.2 + 8 * 0.1 + 7 * 0.1 + 6 * 0.05 + 5 * 0.05 \\
E_b = 10 * 0.1 + 9 * 0.1 + 8 * 0.1 + 7 * 0.1 + 6 * 0.2 + 5 * 0.2 + 0 * 0.2
$$
离散型期望不用积分



## 4. Monte Carlo Integrate

- 随机采样
- 概率密度函数



The restriction to uniform random variables can be relaxed with a small generalization. This is an extremely important step, since carefully choosing the PDF (possibility distribution function) from which samples are drawn is an important technique for reducing *variance* in Monte Carlo (Section 13.10). If the random variables $X_i$ are drawn from some arbitraty PDF $p(x)$, then the estimator
$$
\displaystyle \int_a^bf(x) \ \mathrm dx \ \approx \ \frac{1}{N} \sum_{i = 1}^{N} \frac{f(X_i)}{p(X_i)}  \tag{7.4.1}
$$


以下是式 (7.3.1) 的证明

$f(x)$ 的期望的另一种形式
$$
E_p[f(x)] = \int_D f(x)p(x) \ \mathrm dx \tag{7.4.2}
$$
又对于离散型（趋近于无穷）
$$
E_p[f(x)] \approx \frac{\sum f(x_i)}{N} \tag{7.4.3}
$$
令 $y(x) = \displaystyle \frac{f(x)}{p(x)}$

那么对于半球面上的 BRDF 的积分
$$
\begin{aligned}
\int_\Omega f(x) dx &= \int_\Omega \frac{f(x)}{p(x)} \cdot p(x) dx \\
&= \int_\Omega y(x)p(x) dx \\
&= E_p[y(x)] \\
&\approx \displaystyle \frac{\sum y(x_i)}{N} \\
&= \frac{1}{N} \displaystyle \sum \frac{f(x_i)}{p(x_i)}
\end{aligned} \tag{7.4.4}
$$
当 PDF $p(x)$ 选得比较好的时候，就越接近





## 5. 高维积分

[The Monte Carlo Estimator (pbr-book.org)](https://www.pbr-book.org/3ed-2018/Monte_Carlo_Integration/The_Monte_Carlo_Estimator)

Extending this estimator to multiple dimensions or complex integration domains is straightforward. $N$ samples $X_i$ are taken from a multidimensional (or "joint") PDF, and the estimator is applied as usual. For example, consider the 3D integral
$$
\displaystyle \int_{z_0}^{z_1} \int_{y_0}^{y_1} \int_{x_0}^{x_1} f(x,y,z) \mathrm dx\mathrm dy\mathrm dz \tag{7.5.1}
$$
If samples $X_i = (x_i, y_i, z_i)$ are chosen uniformly from the box from $[x_0, x_1] \times [y_0, y_1] \times [z_0, z_1]$, the PDF $p(X)$ is the constant value
$$
\frac{1}{(x_1 - x_0)}\frac{1}{(y_1 - y_0)}\frac{1}{(z_1 - z_0)} \tag{7.5.2}
$$
and the estimator is 
$$
\frac{(x_1 - x_0)(y_1 - y_0)(z_1 - z_0)}{N} \sum_i f(X_i) \tag{7.5.3}
$$

Note that the number of samples $N$ can be chosen arbitrarily, **regardless of the dimension of the integrand**. This is another important advantage of Monte Carlo over traditional deterministic quadrature techniques. The number of samples taken in Monte Carlo is completely independent of the dimensionality of the integral, while with standard numerical quadrature techniques the number of samples required is exponential in the dimension.

> - deterministic 确定性的
> - quadrature n. 正交；求积

Monte Carlo 是一个线性的方法，和积分的维度无关。使得需要的样本数量与积分的维度无关，而在标准数值积分技术中，需要的样本数量在维度上是指数级的。

如上的例子，是在一个三维空间中进行积分，用蒙特卡洛的方法，均匀随机采样的概率分布函数是 (7.5.2)。

**路径追踪中同样有高维积分，高维积分同样可以用MonteCarlo这样求平均数的方法求解**





## 6. 重要性采样 Importance Sampling

本部分内容，见 [Sampling Random Variables (pbr-book.org)](https://www.pbr-book.org/3ed-2018/Monte_Carlo_Integration/Sampling_Random_Variables)

求 $f(x)$ 在定义域内的积分。

Mente Carlo 积分用随机采样的方式近似积分结果。

样本空间是函数的定义域，随机采样 N 个随机变量 $X_i$ ，$X_i \sim P(X)$, $X_i$ 服从 $P(X)$ 的概率分布。
$$
\displaystyle \int_a^b f(x) \ \mathrm dx \approx \frac{1}{N} \sum_{i = 1} ^{N} \frac{f(X_i)}{p(X_i)} \tag{7.6.1}
$$
样本数越多，结果越接近。



每个样本 $X_i$ 被采样的概率是不一样的，概率分布函数 $p(X_i)$ 不是均匀的，也就是每个 $X_i$ 都有自己的概率。

- PDF 特性1：
  - <font color=red>**在定义域内 $p(X_i)$ 的积分为 1**</font>.

$$
\int _a^b p(x) \mathrm dx = 1
$$

- PDF 特性2：
  - $p(x)$ 和 $f(x)$ 越呈正相关，式 (7.6.1) 的方差越小



### 6.1 逆向采样

因此采样的时候需要带着权重随机，对不同的点的采样是不一样的。这就叫做重要性采样。如

| X    | 1    | 2    | 3    | 4    |
| ---- | ---- | ---- | ---- | ---- |
| P    | 0.2  | 0.5  | 0.1  | 0.2  |
| C    | 0.2  | 0.7  | 0.8  | 1.0  |

如果说样本数有 10 个，那么希望有 2 个采样到 1， 5 个采样到 2,1 个采样到 3， 2 个采样到 4.

在计算机中，用 `random` 函数通常很难进行重要性采样，因为它只能实现一些均匀的随机数。因此要自己实现一种随机数，这种随机数就叫做 **<font color=red>逆向采样</font>**。



**首先，求连续和 C**，就是前 i 个概率 P 的和。

![image-20240710230049803](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407102300828.png)

随机数是可以随机出 0~1 范围内的均匀随机数的，这时候看随机出来的数落在哪个范围。假如 r = 0.5，那么 0.5 落在 > 0.2 且 < 0.7 的地方，也就反推出来选中的是 2. 这个 2 也就是我们真正想要的 随机数

![image-20240710230617413](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407102306436.png)

也就是对于每一个**连续和 C**，可以查找它的每个随机数，即 $C^{-1}(X) = Y$，这个 $Y$ 才是我们真正要的随机变量。





### 6.2 CDF Continious Distribution/Density Function

下面看一个更加复杂的例子，

对于连续型概率分布


$$
CDF = \int_0^x p(x) \mathrm dx
$$
CDF 是单调递增的

抽象的来了，对于连续型，我们每次采样的时候，需要根据计算机提供的 random 函数均匀地在 cdf 上取得一个随机数，然后根据 CDF 得出 $X$

![image-20240710231936780](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407102319811.png)



那么设 $Y = CDF(x)$，则 $Y \in [0, 1]$，由上面的方法，CDF 的反函数
$$
CDF^{-1}(Y) = X
$$
也就是我们真正需要的随机变量

总结起来就是 4 步

1. Compute the CDF $P(x) = \int _0^x p(x') \mathrm dx'$
2. Compute the inverse $P^{-1}(x)$

3. Obtain a uniformly distributed random number $\xi$

4. Compute $X_i = P^{-1}(\xi)$



## 7. Monte Carlo Integrate & Path Tracing

$$
<F^N> = \frac{1}{N} \displaystyle \sum_{i = 0}^{N-1} \frac{f(X_i)}{pdf(X_i)} \tag{7.7.1}
$$

一函数在一定区域内的数值积分可以根据 N 个样本的平均值来估算。本节课将使用 MonteCarlo 积分方法来解决路径追踪中的高维积分。



### 7.1 复习路径追踪



从眼睛出发，看到点 P1，需要计算 P1 点的辐射亮度。在路径追踪中，一条路径会对这个射出的方向做出贡献。路径追踪就是在场景中会有一些路径

![image-20240712184059970](https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407121840988.png)



这条路径能够连接光源，然后找到所有链接光源的路径，把这些路径贡献的辐射亮度累计起来。

场景中的路径可以根据长度分类，长度 n = 1, 2,  3, 4, ..., n

$P_1$ 到 $P_0$ 总的方程可以写为
$$
L(p_1 \rarr p_0)= \sum_{n=1}^\infty P(\bar p_n) \tag{7.7.2}
$$
$p_i$ 就是所有长度为 $i$ 的路径所做出的贡献，它们的和就是所有路径

展开 式(7.7.2)
$$
P(p_2) = \int_A L_e(p_2 \rarr p_1)f(p_2 \rarr p_1 \rarr p_0)G(p_2 \lrarr p_1) \mathrm dA(p_2) \tag{7.7.3}
$$

$$
\begin{aligned}
P(p_3) &= \int_A \int_A L_e(p_3 \rarr p_2)f(p_3 \rarr p_2 \rarr p1) G(p_3 \lrarr p_2) \\ 
& \quad\quad \times f(p_2 \rarr p_1 \rarr p_0)G(p_2 \lrarr p_1)\ \mathrm d A(p_3)\mathrm d A(p_2) + ...
\end{aligned} \tag{7.7.4}
$$

到 $P(p_n)$


$$
P(\bar p_n) = \displaystyle \underbrace {\int_A \int_A ... \int_A}_{n-1} L_e(p_n \rarr p_{n-1}) T(\bar p_n) \mathrm dA(p_2)...\mathrm dA(p_n) \tag{7.7.5}
$$

其中
$$
T_n = \displaystyle \prod_{i=1}^{n-1} f(p_{i + 1} \rarr p_i \rarr p_{i-1})G(p_{i+1} \lrarr p_{i}) \tag{7.7.6}
$$

> 如果需要更具体的推导，见 Ch3



### 7.2 长度为 1，2，...，n

#### 7.2.1 长度为 1

> 注意，这里可能是认为 $P_1 \rarr P_0$ 不算长度，因为是看到的点直接到 camera 的距离，而 $P_2 \rarr P_1$ 才算长度

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407251336588.png" alt="image-20240725133602503" style="zoom:50%;" />

<p style="text-align:center;font-size:smaller">P_2 可以有很多个</p>



- （这时候的点是 $P_0, P_1, P_2$，其中 $P_2$ 是随机的点，也就是随机了 1 个点），这条路径对辐射亮度的贡献 = 光源的辐射亮度 * 这条路径的吞吐量.

    >  光源的辐射亮度是 $L_e(p_2 \rarr p_1)$，吞吐量是 $f(p_2 \rarr p_1 \rarr p_0)G(p_2 \lrarr p_1)$

- 此时的样本空间为场景中的所有点

    <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407251338279.png" alt="image-20240725133818212" style="zoom: 50%;" />

    <p style="text-align:center;font-size:smaller">场景中每个 x 都可以是一个点</p>

#### 7.2.2 长度为 2

- （这时候的点是 $P_0, P_1, P_2, P_3$，其中 $P_2, P_3$ 是随机的），在场景中随机选择两个点 $P_2$ 和  $P_3$，他的 **样本空间** 就是两倍的场景中的点（应该不是 2 倍而是 $n^2$ ?）
- 假如这些点有 1000 个，那么随机选择两个点，样本空间的大小就是 $10000* 9999$，能组合出这么多条路径（假设选过的点不能再选）
- 因为场景中有很多个 $P_2$ 和很多个 $P_3$，所以这是一个二维积分。每找到一条路径都会做一次贡献，那么总共就会有很多路径。



#### 7.2.3 长度为 n-1

- n-1 维积分，要在场景中随机选择 n-1 个点
- 那么样本空间大小有 $10000!$ 
- 蒙特卡洛积分可以用有限条路径的平均值来计算出 $10000!$ 条路径
- 如果用 N 条路径来近似 $10000!$ 条路径，而且要实时，需要 几ms 算完，比如用 100 条路径来算完。接下来就用 MonteCarlo 积分来改写渲染方程



## 9. 路径概率

在改写 MonteCarlo 积分之前，先来做个小游戏

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407251844752.png" alt="image-20240725184412684" style="zoom:33%;" />

青蛙每次跳 1 步，跳到每片叶子的概率相等，且只会往前跳。问青蛙沿着红色线跳的概率？
$$
P = \frac{1}{3} \times \frac{1}{2} \times \frac{1}{2} \times \frac{1}{1}
$$
条件概率，所以是概率连乘。在路径追踪中，也有类似的情况



## 10. 路径追踪 - 重要性采样

### 10.1 去除积分符号

回顾公式 (7.7.2) 以及后续的公式
$$
\begin{aligned}
L(p_1 \rarr p_0) &= L_e(p_1 \rarr p_0) \\
& \quad + \int_A L_e(p_2 \rarr p_1) f(p_2 \rarr p_1 \rarr p_0) G(p_2 \lrarr p_1) \ \mathrm dA(p_2) \\
& \quad + \int_A\int_A L_e(p_3 \rarr p_2) f(p_3 \rarr p_2 \rarr p_1) G(p_3 \lrarr p_2) \\
& \quad \quad \times f(p_2 \rarr p_1 \rarr p_0) G(p_2 \lrarr p_1) \ \mathrm dA(p_3) \mathrm dA(p_2) \quad + \ ...
\end{aligned} \tag{10.1.1}
$$
这里需要把积分符号去掉

回到公式 (7.7.1) 使用 Monte Carlo 方法
$$
<F^N> = \frac{1}{N}\displaystyle \sum_{i=0}^{N-1} \frac{f(X_i)}{pdf(X_i)}
$$

- $f(X_i)$ 

    就是

$$
L_e(p_i \rarr p_{i-1}) T(\bar p_i)\\
$$

$$
T(\bar p_i) = \displaystyle \prod_{i=1}^{n-1} f(p_{i+1} \rarr p_i \rarr p_{i-1})G(p_{i+1} \lrarr p_{i}) \tag{10.1.2}
$$

即每条路径对辐射亮度的贡献

- $pdf(X_i)$
    - 就是随机选择这条路径的概率

总结起来就是，从场景中随机选择 N 条路径，然后求路径对辐射亮度的贡献，除以选择这条路径的概率，最后求平均
$$
\frac{1}{N} \displaystyle \sum_0^{N-1} \frac{L_i}{P_i} \tag{10.1.3}
$$

- 辐射亮度 $L_i$ 
    
- 在前面章节中讲过，等于光源一点的辐射亮度 $L_e$ 乘以整条路径的吞吐量 $T(p_n)$
    
- 概率 $P_i$ 应该怎么算

    - 从前面青蛙跳河的问题可以知道，选择每个点的概率是知道的

        <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407251338279.png" alt="image-20240725133818212" style="zoom: 50%;" />

        如上图中在场景中选择每个点的时候，不能均匀地随机，选择不同的点的概率是不一样的。

        

    - 长度为 1 （$P_2 \rarr P_1$ 才算路径，$P_1 \rarr P_0$ 不算）
        $$
        \frac{1}{N} \displaystyle \sum_{i=0}^{N-1} \frac{L_e(p_2 \rarr p_1)f(p_2 \rarr p_1 \rarr p_0)G(p_2 \lrarr p_1)}{pdf(p_2 \rarr p_1)} \tag{10.1.4}
        $$
        
    - 

    选择多个点的概率是独立随机事件，因此是概率连乘

    - 长度为 2
        $$
        \frac{1}{N} \displaystyle \sum_{i=0}^{N-1} \frac{L_e(p_3 \rarr p_2)f(p_3 \rarr p_2 \rarr p_1)G(p_3 \lrarr p_2) \times f(p_2 \rarr p_1 \rarr p_0)G(p_2 \lrarr p_1)}{pdf(p_3 \rarr p_2) \times pdf(p_2 \rarr p_1)} \tag{10.1.5}
        $$

    - 长度为 n

        就是使用概率连乘

- 



- **总结**

原来公式 (10.1.1) 中的积分
$$
\int_A L_e(p_2 \rarr p_1)f(p_2 \rarr p_1 \rarr p_0)G(p_2 \lrarr p_1) \ \mathrm dA(p_2)
$$
改写为
$$
\frac{1}{N} \displaystyle \sum_{i=0}^{N-1} \frac{L_e(p_2 \rarr p_1) f(p_2 \rarr p_1 \rarr p_0) G(p_2 \lrarr p_1)}{pdf(p_2 \rarr p_1)}
$$
积分
$$
\int_A L_e(p_3 \rarr p_2) f(p_3 \rarr p_2 \rarr p_1) G(p3 \lrarr p_2) \times f(p_2 \rarr p_1 \rarr p_0) G(p2 \lrarr p_1) \ \mathrm dA(p_3) \mathrm dA(p_2)
$$
改写为
$$
\frac{1}{N} \displaystyle \sum_{i=0}^{N-1} \frac{L_e(p_3 \rarr p_2) f(p_3 \rarr p_2 \rarr p_1) G(p3 \lrarr p_2) \times f(p_2 \rarr p_1 \rarr p_0) G(p2 \lrarr p_1)}{pdf(p_3 \rarr p_2) \times pdf(p_2 \rarr p_1)}
$$
长度为 n 的路径同理，主要是去掉积分符号，改为求和符号

而且 Monte Carlo 积分是线性的，和反弹次数无关。无论反弹次数为多少，都不会影响 Monte Carlo 积分



### 10.2 重要性采样

路径追踪从眼睛出发，看到一个点 $p_1$，下一次会从 $p_1$ 出发，看到一些点。

下图中可以看到，在物体表面，法线是 $\bold n$，<font color=red>那么在选择下一个点的时候，就没必要选择背对着的这些点。因为根据渲染方程可以知道，背对着的这写点对辐射亮度的贡献为 0。所以在选择下一个点的时候，只需要向正半球的方向来选择</font>

**也就是把整个场景分为两部分，一部分是正面朝向这一边，另一半是背对着这一边**

<img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407261630262.png" alt="image-20240726163032160" style="zoom:50%;" />

这就是所谓的 **重要性采样**

在随机采样的时候选择更容易产生贡献的点。

那么现在每个点背选择的概率就不一样了，这些背对着的点，被选择的概率为 0

- <font color=red>**重要性采样：选择下一个点的时候，可能根据当前点的几何或者材质来选择下一个点的方向**</font>

    - 假如当前点的平面是一个镜面材质，那么采样的时候就只能往下图中对称出射方向来采样，如下图

        <img src="https://icewalnut-img.oss-cn-shanghai.aliyuncs.com/202407261622611.png" alt="image-20240726162251507" style="zoom:50%;" />

        因为镜子就只能看到那个方向的光，没必要采样其它方向。也就是这个方向的点 概率 $P=1$，其他方向的点的概率为 0.

- 半球面上有非常多的方向，每个方向的概率是不一样的，可以用一个图来表示这个概率分布，如下图，他是一个 pdf 连续函数。每种材质的 pdf 图是不一样的，后面会讲到。 这部分可以配合 "GlobalIllumination.md" 读。














































