# Real Shading in Unreal Engine 4

要求：

* 实时性：高效地同时使用许多可见光源。
* 降低复杂度： 尽可能减少需要的参数，且能够互换使用基于图像的照明和分析光源，所有参数在所有光线类型中始终表现得一致。
* 符合直觉的接口。
* 感知线性：只能为每个像素着色一次。这意味着参数混合着色必须尽可能接近着色结果的混合。
* 易于掌握： 尽量避免对电介质和导体进行技术理解，并尽量减少创建基本物理合理材料所需的工作量。
* 稳定性： 很难错误地创建物理上不可信的材料，所有参数组合都应尽可能稳健和合理。
* 具有描述性： 基本着色模型需要具有足够的描述性，以覆盖现实世界中99%的材质。且所有可分层材质都需要共享同一组参数，以便在它们之间进行混合。
* 灵活性：其他项目和许可证持有者可能不具有相同的真实感目标，因此它需要足够灵活，以实现非真实感渲染。

## 渲染模型

光向量$\bold{l}$以及视角向量$\bold{v}$，依此定义微表面法线：

$$
\bold{h} = \frac{\bold{l} + \bold{v}}{|\bold{l} + \bold{v}|}
$$

### 漫反射BRDF

Burley的漫反射模型，与Lambertian漫射模型相比，只有很小的差异，却无法证明额外的成本是合理的。任何更复杂的漫射模型都很难与基于图像或球面谐波照明一起有效使用。

$$
f(\bold{l}, \bold{v}) =  \frac{c_{diff}}{\pi}
$$

其中$c_{diff}$是材质的漫反射反照率(albedo)。$c_{diff}$往往使用`baseColor`表示。

### 微表面漫反射BRDF

常见的微表面Cook-Torrance模型的一部分：

$$
f(l, v) =  \frac{D(\bold{H})F(\bold{v}, \bold{h})G(\bold{l},\bold{v},\bold{h}))}{4(\bold{n}\cdot\bold{l})(\bold{n}\cdot\bold{v})}
$$

大多数没有以微平面形式具体描述的物理上合理的模型仍然可以被解释为微平面模型，因为它们具有微表面分布函数、菲涅耳因子和一些可以被视为几何阴影因子的附加因子。微平面模型和其他模型之间唯一真正的区别是它们是否包括来自微平面推导的显式$\frac{1}{4cos\theta_lcos\theta_v}$因子。

在实际的微表面模型中，还有一个附加的漫反射项$diffuse$，其代表一个未知形式的函数，通常使用Lambert漫反射来表示，下面简单介绍DFG。

#### Microfacet Distribution Function(D)

**微表面分布函数**通过从测量材料的retroreflective实验中反映，绘制一组曲线并根据峰的高度将材料分为两组，高度被认为是表面粗糙程度，最高峰来自于钢铁，当曲线变得平缓，剩余部分便来自于漫反射。

相较于传统的反射模型，大多数材料在图表中表现的尾部更加长，这意味着从视觉上反射的逸散效果更加绵长，引入GGX基于这种想要更宽尾部的需求，尽管如此，依然无法捕捉到更加明显的辉光。

![Alt](./res/MERL_chrome.png#pic_center)

UE的选择同样基于这样的需求：

$$
D_{GGX}(\bold{h}) = \frac{\alpha^2}{\pi((\bold{n}\cdot\bold{h})^2(\alpha^2-1)+1)^2}
$$

其中$\alpha = Roughness^2$，该方程主要来源于GGX/Trowbridge-Reitz(1975)，并基于Disney的方式重新参数化了$\alpha$，尽管如此，其对于许多材料依然没有足够长的尾部。

简单介绍一下GTR(2012)，Trowbridge和Reitz将他们的分布函数以及其他几种分布与毛玻璃的测量值进行了比较。Berry(1923)的其他分布之一具有非常相似的形式，但指数为1而不是2，导致尾部更长。这表明了一个具有可变指数的更一般的分布，在这里引入，并被称为Generalized-Trowbridge-Reitz，或GTR:

$$
D_{Berry} = \frac{c}{\alpha^2cos^2\theta_h + sin^2\theta_h}
$$

$$
D_{TR} = \frac{c}{(\alpha^2cos^2\theta_h + sin^2\theta_h)^2}
$$

$$
D_{GTR} = \frac{c}{(\alpha^2cos^2\theta_h + sin^2\theta_h)^\gamma}
$$

在上述三类分布函数中，这里的c是一个缩放常量，$\alpha$是一个介于0~1之间用于描述粗糙程度的值，例如0代表一个绝对光滑分布反之1代表完全粗糙或均匀的表面分布。
初步拟合结果表明$\gamma$的典型值在1和2之间。有趣的是，$\gamma=32$的GTR等价于$\theta=2\theta_h$的Henyey-Greenstein相函数；$2\theta_h$的加倍可以看作是将分布从半球扩展到球体。

NDF的抽象定义也许让人感觉难以理解，需要注意的是，微表面发现分布函数，本身并非概率密度函数也不是正态分布函数，而是用于描述一片区域(材料)法线的密度特性。众所周知，一块区域当然并非有有限个法线，法线分布函数自然不能用具体的法线向量来描述，而是对于一片微分区域$dA$，将发现位于立体角$\omega_h$内所有微表面的总面积进行求积：

$$
dA(\omega_h) = D(\omega_h) d\omega_h dA
$$

这个式子的含义是$dA$上法线位于立体角$\omega_i$内所有微表面的总面积，现在总算能理解NDF的本质是一个密度函数。对等式两边基于半球面求积：

$$
\int_H D(\omega_h)d\omega_h dA (\bold{n} \cdot \omega_h) = dA
$$

显然，$dA$项与其他项并没有什么关系，可以约去：

$$
\int_H D(\omega_h)d\omega_h(\bold{n}\cdot \omega_h) = 1
$$

意为：将D投影到宏观表面上，会得到宏观表面的面积，其被约定为1。
Beckmann NDF是光学界(optics community)在第一个微表面模型中使用的法线分布函数。当Cook-Torrance BRDF初步提出的时候也是选用Beckmann，现在同样还在使用。Beckmann 归一化之后的法线分布形式如下：

$$
D(\bold{m}) = \frac{1}{\pi\alpha^2(\bold{n}\cdot\bold{m})^4}e^{\frac{(\bold{n}\cdot\bold{m})^2 - 1}{\alpha^2 (\bold{n}\cdot\bold{m})^2}}
$$

Beckman分布采用了与Blinn-Phong分布非常相似，使用了相同的关系式$\alpha = 2\alpha_p^{-2} -2$，此外，Beckmann分布是满足形状不变性的NDF。

[这里](http://graphicrants.blogspot.com/2013/08/specular-brdf-reference.html)给出了相关的成果整理。

## Fresnel(F)

**菲涅耳反射因子**$F(\theta_d)$表示当光和视角矢量分开时镜面反射的增加，并预测所有光滑表面在掠入射时将接近100%镜面反射。对于粗糙表面，将无法实现100%的镜面反射，但反射率仍将变得越来越高。基本的Schlick近似表示为：

$$
F_{Schlick} = F_0 + (1 - F_0)(1 - cos(\bold{h}, \bold{l}))^5
$$

$$
F_0 = (\frac{\eta_1 - \eta_2}{\eta_1 + \eta_2})^2
$$

$F_0$为正常入射时的镜面反射率。对于电介质是消色差的，对于金属是彩色的（即着色的）。实际值取决于折射率。
因此F取决于$\theta_d$，即光矢量和微法线（即半矢量）之间的角度，而不是与表面法线的入射角。

值得注意的是，掠入射角附近的许多曲线的陡度大于菲涅耳效应预测的陡度。
这一观察结果是Torrance Sparrow(1967)微平面模型解释在更高入射角下出现的“非镜面峰值”的动机。请注意，微平面模型中的$\frac{1}{4cos\theta_lcos\theta_v}$在掠入射角处变得无限大，但这不是问题的原因(无论是在模型中还是在现实世界中)是因为微表面的阴影效应降低了掠反射。

![Alt](./res/Specular_F.png#pic_center)

尽管如此，G与因子的组合也有效放大了菲涅尔效应。
UE4的选择是使用了Schlick's approximation并使用球面高斯近似代替功率：

$$
F(\bold{v}, \bold{h}) = F_0 + (1 - F_0) 2^{-5.55473(\bold{v}\cdot\bold{h})-6.98316(\bold{v}\cdot\bold{h})}
$$

Disney方案对于菲涅尔因子的定义基于其对于物理上实现导致边缘更暗的问题的诉求，通过保证模型在光滑表面的漫菲涅尔阴影和粗糙表面的附加高光之间转换来达到想要的效果，具体来说:对于粗糙表面，光进入和离开围观表面特征的侧面，导致掠入射角处的折射增加。
此外，该方案忽略了漫射菲涅耳因子的折射率，并假设没有入射漫射损失。这允许直接指定入射漫反射颜色。通过Schlick-Fresnel近似，并修改掠射回射响应，使其达到由粗糙度而非零确定的特定值。

$$
f_d = \frac{baseColor}{\pi}(1 + (F_{D90} - 1)(1 - cos\theta_l)^5)(1 + (F_{D90} - 1)(1 - cos\theta_v)^5)
$$

其中：

$$
F_{D90} = 0.5 + 2roughness cos^2\theta_d
$$

也有为了简化计算，令：

$$
f_d = \frac{baseColor}{\pi}
$$

也就是Burly的漫反射公式.
对于光滑表面，该阴影在掠入射角下将入射漫反射率降低0.5，而对于粗糙表面，其响应增加高达2.5。
Schlick版本的菲涅尔公式只能用于描述电介质，例如水、玻璃等，对于导体比如金属等，需要使用参数金属度(metalness)确定金属属性，因此对于非金属的$F_0$进行定义，对于金属的则使用其表面颜色进行定义：

```glsl
F0 = (0.04, 0.04, 0.04)
F0 = mix(F_0, surfaceColor.rgb, metalness)
```

surfaceColor通常使用Albedo，也就是材质的反射率来表示。

## Specular Geometric Attenuation(G)

当光线入射和出射时都会被遮挡一部分，这一部分的功能由**镜面几何衰减**描述，其用于计算最后出射的光线占据入射光线的比例:
在测量数据中分离G是困难的，因为它需要D和F因子的精确估计以及镜面与漫射的分离。然而，G对方向反照率(Albedo)的影响可以间接地看出。反照率是总反射能量与总入射能量的比值。因此，对于所有角度和波长，反照度都必须小于1。

大多数材料的方向反照率在前70度相对平坦，掠入射角的反照率与表面粗糙度密切相关。光滑的材料在75度左右略有增加，随后向90度下降。粗糙的表面通常会显著增加。值得注意的是，反照率值总体上相当低，少有材料的反照率超过0.3。

倘若完全忽略G和$\frac{1}{cos\theta_lcos\theta_v}$因子，成为"无G"模型，会导致掠入射角的相应过暗。
因此我们可以得知：G函数的选择对反照率有着深远的影响，反照率反过来又对表面外观有着深远影响。

$$
k = \frac{(Roughness + 1)^2}{8}
$$

$$
G_1(\bold{v}) = \frac{\bold{n}\cdot\bold{v}}{(\bold{n}\cdot\bold{v})(1-k) + k}
$$

$$
G(\bold{l}, \bold{v}, \bold{h}) = G_1(\bold{l})G_1(\bold{v})
$$

使用Schlick模型并令$k = \alpha/2$以更好拟合GGX的Smith模型，在这种情况下$\alpha = 1$时Schlick与Smith完全匹配，并在其他状况下相当近似。在此基础上，使用$(Roughness+1)/2$以代替原有粗糙度用以降低热度，这样做的优点是能够尽量保证提升掠角的亮度。

继续看一看Disney的渲染方案：

通过使用GGX为Walter方案的G，并重新映射粗糙度以减少光泽表面的极端增益：

$$
\alpha_g = (0.5 + \frac{roughness}{2})^2
$$

由此，将原始粗糙度从$[0，1]$范围线性缩放到缩小的范围$[0.5，1]$。

## Image Based Lightning

IBL相当于一个无限大的球面光源在照射场景。辐射度由生成的一个Environment Map决定。

### 低差异(Discrepancy)序列

首先给出Discrepancy的概念：

$$
D_N(P) = sup_{B\in J}|\frac{A(B)}{M} - \lambda_S(B)|
$$

对于一个在$[0, 1]^n$空间中的点集，任意选取一个空间中的区域B，此区域中的点的数量A和点集个数的总量N的比值和此区域的体积$\lambda_s$的差的绝对值的最大值，就是这个点集的Discrepancy。分布越均匀的点集，任意区域内的点集数量占点总数量的比例也会越接近于这个区域的体积。

生成序列的必经之路是通过Radical Inversion运算：

$$
i = \sum^{M-1}_{l=0} a_l(i)b^l
$$

$$
\Phi_{b,C}(i) = [b^{-1}...b^{-M}][C(a_0(i)...a_{M-1}(i))^T]
$$

b是一个正整数，对于任何一个整数i，将其表示为b进制的数，再将得到的数中的每一位上的数字$a_l(i)$左乘一个生成矩阵C生成一个新的向量。
再将得到的向量组合成的数镜像到小数点右边，就能够得到这个数以b为底数和以C为生成矩阵的Radical Inversion。

令C等于单位矩阵，这就是Van der Corput序列的定义，具体公式就不写了，反正用不着。

再稍微拓展一下，介绍一下Halton序列和Hammerslay点集：
Halton序列在Van der Corput序列基础上进行了一个微小的改动，将生成的每一个数使用互质的b作为基底：

$$
X_i \coloneqq (\Phi_{b_1}(i),...,\Phi_{b_{n-1}}(i))
$$

每个b之间互为质数。

在Halton的基础上，Hammerslay点集只能生成固定样本数量，并且将第一个维度变成了$\frac{i}{N}$，i也就是样本点的索引，N为样本点集中点的个数。

$$
X_i \coloneqq (\frac{i}{N},\Phi_{b_1}(i),...,\Phi_{b_{n-1}}(i))
$$

Hammersley的Discrepancy比Halton更稍低一些，但是代价是必须预先知道点的数量，并且一旦固定没法更改。Halton序列的一个缺点是，在用一些比较大的质数作为底数时，序列的分布在点的数量不那么多的时候并不会均匀的分布，只有当点的数量接近底数的幂的时候分布才会逐渐均匀。
UE4中对环境贴图的Filter采样就是用的点集大小固定为1024的Hammersley点集。

看到这里已经知道了如何生成，并且起到了什么样的作用，可以暂时回到主视角，以后有兴趣可以再作拓展。

### 漫反射

通过生成低差异序列，将一个区域的积分转化为一个点集的累加计算，这是符合计算机计算性能优化方案的。
先从漫反射公式开始，也是基于Cook-Torrance的BRDF反射公式：

$$
L_o(p, \bold{v}) = \int_\Omega (k_d \frac{c}{\pi} + \frac{D F G}{4(\bold{n}\cdot\bold{l})(\bold{n}\cdot\bold{v})})L_i(p, \omega_i)(n \cdot \omega_i) d\omega_i
$$

其中$k_d = (1 - F)(1 - metallic)$，因而可以知道出射Irradiance通过albedo、roughness、metalness三个参数来控制，这三个参数体现在texture上，作为材质传递进入renderpass。
漫反射的部分是括号内的第一项，在反射模型中，假定环境光来源于无穷远处，类比于天空盒：

$$
L_o(p, \bold{v}) = \int_\Omega k_df_dL_i(p, \bold{l})(n\cdot\bold{l})d\omega_i
$$

我们知道，$k_d$包含一个F因子，其需要一个$cos(h, \bold{v})$，h与$\omega_i$息息相关，因此如果想要提取$k_d$就需要做一个近似计算，先说结论：

$$
F^* = F_0 + (\max(1 - roughness, F_0) - F_0)(1 - cos(n, v))^5
$$

使用法线向量n代替了中线向量h，并对于$\max(1 - roughness, F_0)$替换了1。
进行进一步优化积分，通过拟蒙特卡洛积分对环境贴图进行预积分生成，实时渲染时通过片元法线采样：
经过优化，便有了：

$$
G(\bold{v},\bold{l},\bold{h},\alpha) = (1 - F^*)(1 - metallic)\cdot albedo \cdot \frac{\sum_i^N L_i(\omega_i)}{N}
$$

这是使用余弦权重的半球采样：

$$
pd f = \frac{cos \theta_l}{\pi}
$$

即使具有重要性采样，仍然需要获取许多样本。通过使用mip映射可以显着减少样本计数，但对于足够的质量，计数仍然需要大于16。
需要补充的是对于漫反射优化的整体框架，预留以方便未来列举其他的方式：

$$
\int_H L_i(\bold{l})f(\bold{l}, \bold{v}) cos \theta_l d\bold{l} \approx \frac{1}{N} \sum^N_{k=1} \frac{L_i(\bold{l_k})f(\bold{l}, \bold{v})cos\theta_{\bold{l}_k}}{p(\bold{l}_k, \bold{v})}
$$

### 镜面反射

镜面反射自然就是Cook-Torrance BRDF模型的剩下一部分：

$$
L_o(p, \bold{v}) = \int_{\Omega} f_r L_i(p, \bold{l})(\bold{n}\cdot\bold{l}) d\omega_i
$$

$$
f_r = k_sf_s = \frac{DFG}{4(\bold{n}\cdot\bold{l})(\bold{n}\cdot\bold{v})}
$$

UE提出的方式**Split Sum Approximation**是将该积分拆成两部分进行计算，然后单独预先计算每部分的求和：

$$
\frac{1}{N} \sum^N_{k=1}\frac{L_i(\bold{l}_k) f(\bold{l}_k, \bold{v}) cos\theta_{\bold{l}_k}}{p(\bold{l}_k, \bold{v})} \approx \pod{\frac{1}{N} \sum^N_{k=1} L_i(\bold{l}_k)}(\frac{1}{N}\sum^N_{k=1} \frac{f(\bold{l}_k,\bold{v})cos\theta_{\bold{l_k}}}{p(\bold{l}_k, \bold{v})})
$$

该式需要四个参数法线n, 视角向量v, $F_0$以及粗糙度roughness。
在使用拟蒙特卡洛进行优化之前，应当保证拆分前后的结果尽可能相似，两个被积式需要满足两点条件：

1. 该函数的support尽可能小，即对应的积分区域。
2. 函数值变化频率相对低频，换句话说也就是smooth。

尽管有误差，作为环境光来说是可以忽略的！

我们将左边一部分称为**Prefilter Environment Map**，右边称为**Lut Map**。接下来分开介绍：

预先计算不同粗糙度值的第一个总和，并将结果存储在立方体映射的mip-map级别，这是常见的做法，AMD基于此提供了相关的工具。

#### Prefilter Environment Map

让我们看回积式，首先观察**Prefilter Environment Map**，我们使用$L_c$代替：

$$
L_o(p, \bold{v}) = L_c \cdot \int_\Omega f_r (\bold{n} \cdot \bold{l}) d\omega_i
$$

想要将$L_c$从积分中剥离，免不了一番斗争：

$$
L_c = \frac{\int_\Omega f_r L_i(p, \bold{l}) (\bold{n} \cdot \bold{l}) d\omega_i}{\int_\Omega f_r (\bold{n} \cdot \omega_i)d\omega_i}
$$

上文已经提到了，$\int_H D(\omega_h)d\omega_h(\bold{n}\cdot \omega_h) = 1$，此外基于Cook-Torrance BRDF的推导：

$$
\frac{\omega_h}{\omega_i} = \frac{1}{4(\omega_o \cdot\omega_h)}
$$

有：

$$
pdf(\omega_i) = \frac{D(\omega_h)(\bold{n}\cdot\omega_h)}{4(\omega_o \cdot\omega_h)}
$$

让我们从头开始，代入进行拟蒙特卡洛积分，进行优化：

$$
L_c = \frac{\frac{\frac{1}{N}\sum^N_k f_r L_i(p, \omega_i)}{pdf(\omega_i)}}{\frac{\frac{1}{N}\sum_k^N f_r (\bold{n}\cdot \omega_i)}{pdf(\omega_i)}} = \frac{\frac{\frac{1}{N}\sum^N_k D(\omega_h) F(\omega_h, \omega_o) G(\omega_i, \omega_o)  L_i(p, \omega_i)(\bold{n}\cdot \omega_i)}{4 (\bold{n}\cdot\omega_i)(\bold{n}\cdot\omega_o)pdf(\omega_i)}}{\frac{\frac{1}{N}\sum_k^N D(\omega_h) F(\omega_h, \omega_o) G(\omega_i, \omega_o) (\bold{n}\cdot \omega_i)}{4 (\bold{n}\cdot\omega_i)(\bold{n}\cdot\omega_o)pdf(\omega_i)}} = \frac{\frac{\frac{1}{N}\sum^N_k D(\omega_h) F(\omega_h, \omega_o) G(\omega_i, \omega_o)  L_i(p, \omega_i)(\bold{n}\cdot \omega_i)}{(\bold{n}\cdot\omega_h)}}{\frac{\frac{1}{N}\sum_k^N D(\omega_h) F(\omega_h, \omega_o) G(\omega_i, \omega_o) (\bold{n}\cdot \omega_i)}{(\bold{n}\cdot\omega_h)}}
$$

进一步地，我们需要针对F项进行优化，老生常谈的Schlick优化：

$$
F(\bold{n}, \bold{l}) \approx F_0 + (1 - F_0)(1 - (\bold{n} \cdot \bold{l})^+)^5
$$

F可以直接约掉了，基于两个现象：

1. 当物体光滑时，$\omega_h$和$\omega_o$是十分接近的，从上面的公式来看，这时候的F项更接近一个定值。
2. 当物体相对粗糙时，$L_c$本身的值也是相对小的，所以这个近似也是接受的。

同时，因为是镜面反射，很容易作出这样的假设$\omega_o = \bold{n} = R$, 也就是镜面反射BRDF的Lobe只会在反射方向$\omega_o$附近，光凭文字可能有些难以理解，简而言之，以Lobe的方向，或者说$\omega_o$为法线方向建立一个新平面，自然，$\omega_o$的方向等同于Lobe的方向等同于法向量$\bold{n}$的方向。
既然$\bold{n}$，$\bold{v}$都一样了，就可以开始对G进行优化了。
在此基础上，我们知道$G(\bold{l}, \bold{v}, \bold{h}) = G_1(\bold{h}, \bold{l})$:

$$
L_c = \frac{\frac{1}{N}\sum_k^N G_1(\omega_i, \omega_o) L_i(p, \omega_i)}{\frac{1}{N}\sum_k^N G_(\omega_i, \omega_o)}
$$

UE选择使用$W(\omega_i) = \bold{n} \cdot \omega_i$代替$G_1$

实际的是使用中先通过roughness确定对应的mipmap层级，再通过对应反射方向$R$来获取对应的$L_c$项。


基于上述理论，附上具体的HLSL实现：

```hlsl
float3 ImportanceSampleGGX( float2 Xi, float Roughness , float3 N ) 
{ 
  float a = Roughness * Roughness; 
  float Phi = 2 * PI * Xi.x; 
  float CosTheta = sqrt( (1 - Xi.y) / ( 1 + (a*a - 1) * Xi.y ) ); 
  float SinTheta = sqrt( 1 - CosTheta * CosTheta ); 
  float3 H;
  
  H.x = SinTheta * cos( Phi ); 
  H.y = SinTheta * sin( Phi ); 
  H.z = CosTheta; 
  
  float3 UpVector = abs(N.z) < 0.999 ? float3(0,0,1) : float3(1,0,0); 
  float3 TangentX = normalize( cross( UpVector , N ) ); 
  float3 TangentY = cross( N, TangentX );

  // Tangent to world space 
  return TangentX * H.x + TangentY * H.y + N * H.z; 
}

float3 SpecularIBL( float3 SpecularColor , float Roughness , float3 N, float3 V )
{ 
  float3 SpecularLighting = 0; 
  const uint NumSamples = 1024; 
  for( uint i = 0; i < NumSamples; i++ ) 
  { 
    float2 Xi = Hammersley( i, NumSamples );
    float3 H = ImportanceSampleGGX( Xi, Roughness , N ); 
    float3 L = 2 * dot( V, H ) * H - V; 
    float NoV = saturate( dot( N, V ) ); 
    float NoL = saturate( dot( N, L ) ); 
    float NoH = saturate( dot( N, H ) ); 
    float VoH = saturate( dot( V, H ) );

    if( NoL > 0 ) 
    { 
      float3 SampleColor = EnvMap.SampleLevel( EnvMapSampler , L, 0 ).rgb; 
      float G = G_Smith( Roughness , NoV, NoL ); 
      float Fc = pow( 1 - VoH, 5 ); 
      float3 F = (1 - Fc) * SpecularColor + Fc; 
      // Incident light = SampleColor * NoL 
      // Microfacet specular = D*G*F / (4*NoL*NoV) 
      // pdf = D * NoH / (4 * VoH) 
      SpecularLighting += SampleColor * F * G * VoH / (NoH * NoV); 
    } 
  }
  return SpecularLighting / NumSamples; 
}
```