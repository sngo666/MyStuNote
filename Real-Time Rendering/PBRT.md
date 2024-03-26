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

$r$表示从光源到点p处的距离，$\theta$表示微表面(光源)的法线和出射到点p之间的光线形成的夹角。这一转换可以被看作是，将这个表面的光照强度折损成朝向点p的强度，并折损距离上的损失(想象球壳捕获能量，与球壳面积成比例，因此是$r^2$)
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