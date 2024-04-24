# PBRT

## Radiometry

### 基础量

* Energy

能量以joules(J)作为单位，光子自光源发射而出，携带能量且具有特定的波长：

$$
Q = \frac{hc}{\lambda}
$$

$c$代表光速，299,472,458m/s.
$h$代表普朗克常数，$h \approx 6.626 \times 10^{-34}m^2kg/s$

* Flux(lm)

通量表示单位时间内通过区域或者空间的能量，取微分时间内能量的极限，求出辐射通量。
顾名思义，其单位为$J/s$，也就是watts(W)

$$
\Phi = \lim_{\bigtriangleup t \rightarrow 0} \frac{\bigtriangleup Q}{\bigtriangleup t} = \frac{dQ}{dt}
$$

对于Flux的通俗描述可以理解成一个将光源完全包含的球体，在每个单位时间(对于计算机渲染过程来说)，该球面接收到光源发出的所有光子，以此作为描述通量的标准。当然，这也可以不是个球体，因为Flux本身并不会描述空间内任何距离或是区域上的特性。它仅仅代表光源在一个时间单位发出的全部能量。

* Irradiance(lx)

Irradiance(辐照度) 和 Radiance(辐射度)分别描述Flux的方向特性以及方向和面积特性(粗浅理解)。

依然以环光源球体为例，假设两个不同层次(半径)的同心球体，光源位于球心。对于外层球体的单位区域($dA$)，单位时间内通过的光子数量必定小于内层球体单位区域上单位时间的光子通量。
道理很简单：每个单位时间内，光子(能量)是固定的，半径越大，表面积就越大，区域能捕获的光子就越少。

现在量化这个假设：

$$
E = \frac{\Phi}{4\pi r^2}
$$

对于一个微分区域A来说：

$$
E(p) = \lim_{\bigtriangleup A \rightarrow 0} \frac{\bigtriangleup \Phi(p)}{\bigtriangleup A} = \frac{d \Phi{p}}{dA}
$$

$p$用来代表一个点，因此$E(p)$用于描述点p每微分面积的辐照度(直射情况下)。
另一方面，光线到达物体表面的强度与物体表面和光线之间夹角的余弦成正比：

$$
E_2 = \frac{\Phi \cos \theta}{A}
$$

尽管我们不会主动提及Irradiance反映了光源在距离上的能量属性，因为这太片面了，而是用一个点上的微分面反应光源在该点上的能量属性，尽管可能理解上会比较笼统，但是定义上非常全面。

* Intensity(cd)

当我们聚焦在光源上一个固定的角度散射的光子，这片光子会随着距离的拉长而散射在更大的区域中，但不变的是一旦角度固定，光子的能量是守恒的。这个角度被定义为立体角，是一个微分的角度，用于描述一个光源在某个方向上的强度。
可以想象：这个放射出去的光线呈一个微妙的锥形。

$$
I = \lim_{\bigtriangleup \omega \rightarrow 0} \frac{\bigtriangleup \Phi}{\bigtriangleup \omega} = \frac{d\Phi}{d\omega}
$$

虽然强度描述了方向的意义，但是和上面所假设的模型一样，这些只是用于描述并量化光源本身的性质，与环境并没有任何关系。

* Radiance(nit)

辐亮度(Radiance)是描述光源的最重要的属性。在上文中，Intensity和Irradiance分别描述了光源在微分方向和微分区域上的能量性质，当然，也都是建立在微分时间单位基础上的。Radiance完成了这最后一步的组合。
可以说，Radiance是Intensity在描述光源在固定方向微分角的基础上，进一步描述在特定微分区域上的能量强度；也是Irradiance在微分区域上的能量性质的基础上，加入特定微分立体角的性质。所以，可以这么写：

$$
L(p, w) = \frac{d^2 \Phi}{d\omega dA^\perp} = \frac{dE_{\omega}(p)}{d\omega} = \frac{dI(\omega)}{dA \cos\theta}
$$

辐亮度是最基本的用于描述光强的属性，当给出辐亮度，意味着光源在特定方向和特定表面上的光强性质。另一方面，它能够保证光线在穿过真空时保持稳定。

* 入射和出射光照函数

光与场景中的表面进行互动时，辐照函数在表面上往往不是连续的，对于表面的外层和里层也有可能是完全不同的。

使用$L_i(p, \omega)$描述光源出射点p的Radiance，函数的箭头朝向并不重要，一般来说，都是指向光源的。
使用$L_o(p, \omega)$描述在点p射出的Radiance。而对于表面的反面，则使用$-\omega$作为输入参数。

简单来说，这些函数只描述单面的性质。需要注意的是：

$$
L_o(p, \omega) = L_i(p, -\omega) = L(p, \omega)
$$

即：

$$
L^+ = L^-
$$

* 光谱分布

到目前为止，所有辐射量的定义都没有考虑在波长上分布的变化，将光谱亮度$L_{\lambda}$定义为在无限削的波长间隔内的亮度极限$\bigtriangleup \lambda$

$$
L_{\lambda} = \lim_{\bigtriangleup \lambda \rightarrow 0} \frac{\bigtriangleup L}{\bigtriangleup \lambda} = \frac{dL}{d\lambda}
$$

所有这些光谱变体的单位都有一个附加系数$1/m$。
通过对描述人眼对于各种波长的相对灵敏度的光谱响应曲线V(A)积分，可以将每个光谱辐射量转换为响应的光度量。
最后我们描述亮度Y:

$$
Y = \int_{\lambda} L_{\lambda}(\lambda)V(\lambda) d\lambda
$$

## Radiometric Integrals

表示法向为n的点p处在一组方向$\Omega$的辐照度为：

$$
E(p, \bold{n}) = \int_{\Omega} L_i(p, \omega) |\cos \theta| d\omega
$$

通常而言，这个方向集合代表表面法线所朝向的半球面。

辐射量积分中的各种余弦因子常常会分散掉积分所想象要表达的内容(我觉得这句话就是字面上的意思，精简表达):

$$
E(p, \bold{n}) = \int_{H^2(\bold{n})} L_i(p, \omega) d \omega^{\perp}
$$

* 关于立体角

具体证明不多作说明，大概就是从球心出发，在球面上放射一道光线，这个光线与z轴呈现一个$\theta$角，并转动一个$d\theta$角，与此同时，在xOy平面转动一个$d\phi$角度，在球面上变动前后两个点作为对角的顶点形成一个近似矩形(这样思考)。因此根据两条边长求解面积。

$$
d\omega = \sin \theta d\theta d\phi
$$

因此，将入射能量表示为：

$$
E(p, \bold{n}) = \int_{0}^{2\pi} \int_{0}^{\pi / 2} L_i(p, \theta, \phi) \cos \theta \sin \theta d\theta d\phi
$$

* 面光源

假设存在一个输出恒定的四边形，计算其在p点处得到的辐照度,我们将方向微分转换为面积微分：

$$
d\omega = \frac{dA\cos\theta}{r^2}
$$

$r$表示从光源到点p处的距离，$\theta$表示微表面(光源)的法线和出射到点p之间的光线形成的夹角。这一转换可以被看作是，将这个表面的光照强度折损成朝向点p的强度，并考虑距离上的损失(想象球壳捕获能量，与球壳面积成比例，因此是$r^2$)
这个式子清晰反映了Intensity和Irradiance之间的关系，可见Irradiance相较于前者，性质上更”靠近“Radiance。

$$
E(p, \bold{n}) = \int_A L\cos \theta_i \frac{\cos\theta_o dA}{r^2}
$$

这里反映了一个表面射出能量到另一个表面的过程，探讨的模型不再仅仅限于一个表面和一个纯粹放射能量的光源。有这一基础，场景中的光照基本由这两种情况组成，光源到表面，再由表面到表面，然而这只是最简单的讨论。

## Surface Reflection

描述光反射机制有两个抽象：BRDF与BSSRDF
BRDF以相对简单的方式描述了地表反射，而后者概括了前者，并描述了半透明材料的光反射机制。

### BRDF

假设一束光线从方向角度$\omega_i$进入p点(所在的微表面，下略)，并在出射方向$\omega_o$被摄像机捕获，可以说这里的Irradiance被表示为：

$$
dE(p, \omega_i) = L_i(p, \omega_i) \cos \theta_i d\omega_i
$$

鉴于几何光学的线性假设(通俗理解为，入射的能量以一定比例损失)，可以表示为：

$$
f_r(p, \omega_o, \omega_i) = \frac{dL_o(p, \omega_o)}{L_i(p, \omega_i)\cos\theta_id\omega_i}
$$

对于$f_r$的理解要基于一个概念，就是其只反映一个入射方向与出射方向的Radiance的比例，并不代表出射方向的Radiance，也就是说，仅仅代表出射方向的一个微分，表面出射方向的Radiance应当由所有入射方向的Irradiance共同决定(在BRDF中)，这一点能够更好地理解公式的分子为什么是一个Radiance的微分。最后，如果对于为什么是Irradiance有疑问，答案很简单：因为入射点p。
需要特别说明的是，BRDF遵循两个特质：

1. 对于所有的入射和出射方向,$f_r(p, \omega_i, \omega_o) = f_r(p, \omega_o, \omega_i)$
2. 反射光的能量总是小于等于入射光的能量：

$$
\int_{H^2(\bold{n})} f_r(p, \omega_o, \omega') \cos \theta' d\omega' \le 1
$$

对于半球上指定出射方向，我们对于所有入射方向的Irradiance和BRDF函数进行积分计算，得出该表面的半球反射率:

<span id="rhohd">

$$
\rho_{hd}(\omega_o) = \int_{H^2(\bold{n})} f_r(p, \omega_o, \omega_i) |cos\theta_i|d\omega_i
$$

</span>

这个特质同时表明了另一件事：在固定的某个入射和出射方向上，其BRDF并不一定小于等于1.
关于BRDF最后要表示是，对于hemispherical-hemispherical reflectance，也就是半球-半球反射率，对于微表面所在半球面，分别考虑所有入射方向和所有出射方向的积分，以求解$\rho_{hh}$:

$$
\rho_{hh} = \frac{1}{\pi} \int_{H^2(\bold{n})}\int_{H^2(\bold{n})}f_r(p, \omega_o, \omega_i) |\cos \theta_o\cos\theta_i|d\omega_i d\omega_o
$$

bidirectional transmittance distribution function(BTDF)描述了透射光的分布，使用$f_t(p, \omega_o, \omega_i)$，需要注意的是，$\omega_i$与$\omega_o$位于点p周围的相反半球，此外，BTDF不满足上述的互易性。

bidirectional scattering distribution function(BSDF)是BRDF与BTDF相结合的产物：

$$
dL_o(p, \omega_o) = f(p, \omega_o, \omega_i) L_i(p, \omega_i) |cos\theta_i| d\omega_i
$$

最后，鉴于透射方程的加入，可以完成基于整个球体的反射方程：

$$
L_o(p, \omega_o) = \int_{S^2}f(p, \omega_o, \omega_i) L_i(p, \omega) |cos \theta_i| d\omega_i
$$

### BSSRDF

bidirectional scattering surface reflectance distribution function (BSSRDF)用于描述亚表面光传输的材料散射形式，对于分布函数$S(p_o, \omega_o, p_i, \omega_i)$描述了入射点$p_i$以及角度$\omega_i$的Flux在对应的出射点恶化出射角度的Radiance之间的关系。

$$
S(p_o, \omega_o, p_i, \omega_i) = \frac{dL_o(p_o, \omega_o)}{d\Phi(p_i, \omega_i)}
$$

很明显，随着两点之间距离的递增，$S$也许会以一个非常陡峭的倍率下降。
BSSRDF与BRDF最大的区别在于，前者发生在一个点上，而后者明显不是，这意味着积分运算会复杂许多：

$$
L_o(p_o, \omega_o) = \int_A\int_{H^2(\bold{n})} S(p_o, \omega_o, p_i, \omega_i)L_i(pi, \omega) |\cos\theta_i| d\omega_idA
$$

在计算云或者烟雾中的光线传播基于同样的效应。

## Light Emission

原子在高于绝对零度的情况下运动，带有电荷的原子粒子运动导致发射一定波长的电子辐射，基于此诞生了许多不同种类的光源。

Luminous efficacy(光效)衡量了将功率转化为可见光的效率，对于人类来说，不可见光几乎没有价值。其表现为发射的光度量和辐射量(在所有波长发射的总功率)的比率：

$$
\frac{\int \Phi_e(\lambda) V(\lambda) d\lambda}{\int\Phi_i(\lambda)d\lambda}
$$

$V(\lambda)$表示光谱响应曲线。
光效也可以被定义为表面上某一点的发光亮度与Irradiance的比值。

### Blackbody Emitters

黑体表示为尽可能有效的将能量转换为电磁辐射，这在现实中不可能被实现，但是可以用来表示发射波长和温度的函数关系。黑体意味着完美吸收了所有的入射能量，并不反射任何能量，普朗克定律给出了黑体发出的辐射率和波长的函数关系：

$$
L_e(\lambda, T) = \frac{2hc^2}{\lambda^5(e^{hc/\lambda k_bT}-1)}
$$

$k_b$表示玻尔兹曼(Boltzmann)常数$1.3806488\times10^{-23}J/K$，$K$表示温度。理想状态下，黑体向所有方向均匀辐射。

随着温度的升高，更多的发射光处于可见频率内(380nm至780nm之间)，并且发射总能量有迅猛的增长。
最后，要注意单位统一。

* 非黑体辐射

基尔霍夫定律描述了非黑体的辐射，任何频率下的发射Radiance分布等于完美黑体在该频率下的Radiance乘以在该频率上被物体吸收的入射辐射的分数，吸收的辐射率等于1减去反射的量。

$$
L'_e(T, \omega, \lambda) = L_e(T, \lambda)(1 - \rho_{hd}(\omega))
$$

普朗克定律给出了$Le$，$\rho_{hd}(\omega)$给出了半球方向的反射率，正是[前文](#rhohd)已经给出的。

Stefan–Boltzmann定律给出了在点p的辐射出射度：

$$
M(p) = \sigma T^4
$$

$\sigma$代表了Stefan–Boltzmann常数$5.67032\times Wm^{-2}K^{-4}$
可以注意到，所有频率上的总发射以$T^4$的速率倍增，这意味着温度加倍会使发射的总能量增加16倍。

如果发射器的发射光在某一个温度下与黑体分布相似，可以说发射器具有相应的色温。找到具体色温的方法取发光器最高的波长，使用Wien’s displacement定理给定温度下合体发射最大值的波长：

$$
\lambda_{max} = \frac{b}{T}
$$

$b$代表了Wien’s displacement常数 $2.8977721 \times 10^{-3}mK$
白炽灯的色温一般在2700K左右，卤钨灯的色温在3000K左右，荧光灯的亮度范围在2700K到6500K之间，温度超过5000K被描述为冷，而2700~3000K被描述为暖。

* 其他标准

国际委员会CIE制定了另外一套对于光发射分布的定义。
标准光源A对应于黑体辐射体约2856K。
其他的在有需要时再作补充。

## Representing Spectral Distributions

现实中的光谱会很复杂，为了能够正确表示包含各种复杂光谱的场景图像，必须要让渲染器有效准确表示光谱分布，从定义常数到表示光谱，使用nm作为表示波长的基本单位。
一般来说，pbrt只会关注肉眼能够捕捉到的光谱波长，也就是大约400~380之间。可以使用一组基函数和对应的系数去表示SPD(Spectral power distributions)，也可以使用一组离散的点，并藉由线性插值完成SPD的存储和再现。

## color

颜色感知的三刺激理论认为，所有可见光谱分布都可以使用三个标量值准确地表示给人类观察者。其基础是眼睛中有三种类型的感光视锥细胞，每种细胞对不同波长的光敏感。

对于光谱分布$S(\lambda)$和三个刺激匹配函数$m_{1,2,3}(\lambda)$相乘得到相应的刺激值$v_i$

$$
v_i = \int S(\lambda)m_i(\lambda)d\lambda
$$

匹配函数给出了一个颜色空间，通过三刺激值的3D向量空间：两个光谱之和的三刺激值由它们的三刺激值和与已缩放的光谱相关联的三刺激值之和给出。
给定一个光谱分布函数$S(\lambda)$，其xyz色彩空间坐标通过与对应的光谱匹配曲线的积来计算

$$
x_\lambda = \frac{1}{\int_\lambda Y(\lambda)d\lambda}\int_\lambda S(\lambda)X(\lambda) d\lambda\\
y_\lambda = \frac{1}{\int_\lambda Y(\lambda)d\lambda}\int_\lambda S(\lambda)Y(\lambda) d\lambda\\\
z_\lambda = \frac{1}{\int_\lambda Y(\lambda)d\lambda}\int_\lambda S(\lambda)Z(\lambda) d\lambda\\
$$

CIE $Y(\lambda)$三刺激曲线与用于定义光度量的$V(A)$光谱响应曲线成比例：

$$
V(A) = 683 Y(A)
$$

对于两个光谱的简单1D积分，通过黎曼积分来计算：

$$
\int^{\lambda_{max}}_{\lambda_{min}} f(\lambda)g(\lambda) \approx \sum^{\lambda_{max}}_{\lambda = \lambda_{min}} f(\lambda)g(\lambda)
$$


