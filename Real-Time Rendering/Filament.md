# Filament Document

## 闲言几句

Filament是一款基于物理的安卓渲染（PBR）引擎。其目标是为Android开发人员提供一套工具和应用程序接口，使他们能够轻松创建高质量的2D和3D渲染。
该文档旨在解释Filament中使用的材质和光照模型背后的方程式和理论，而并非引擎的实现与代码层面的设计。并旨在为Filament的贡献者或对引擎内部运作感兴趣的开发者提供参考。

## 法则

与UE中的PBR一样，首先花一点点时间确定开发目标：

1. 实时移动性能：该引擎主要面向移动平台渲染系统。
2. 质量：质量尽量服务于不同配置的硬件设备。
3. 易用性：服务于可快速迭代的艺术资产。
4. 直觉性：使用公制物理单位。
5. 部署规模：尽量保证精简。

选择PBR的原因在于，与传统的实时模型相比，它能更准确地呈现材质及其与光线的相互作用。此外，PBR 方法的核心是将材质和光照分离，这样就能更轻松地创建在所有光照条件下都能准确呈现的逼真资产。

## 材料系统

本节将会描述多种材料模型，以简化对各向异性或透明涂层等各种表面特征的描述。但在实际应用中，其中一些模型被浓缩为一个模型。例如，标准模型、透明涂层模型和各向异性模型可以组合成一个更灵活、更强大的模型。

### 标准模型

材料模型由双向散射分布函数 BSDF（Bidirectional Scattering Distribution Function）进行数学描述，BSDF 本身由另外两个函数组成：双向反射分布函数 BRDF（Bidirectional Reflectance Distribution Function）和双向透射分布函数 BTDF（Bidirectional Transmittance Function）。
这些都是老生常谈了(具体见PBRT相关章节)

标准材料模型将侧重于BRDF，而忽略BTDF，或对其进行极大的近似。因此标准模型只能正确模拟反射、各向同性、介电或平均自由路径短的导电表面。
BRDF将标准材料的表面响应描述为由两个项组成的函数：

漫反射部分$f_d$(diffuse)
镜面反射部分$f_r$(specular)

![Alt](./res/filament_standard_model.png#pic_center)

将二者结合，变成完成的表面反射，这就是一直沿袭的表面反射框架：

$$
f(\bold{v}, \bold{l}) = f_d(\bold{v}, \bold{l}) + f_r(\bold{v}, \bold{l})
$$

显然，完整的渲染方程需要对于整个半球进行积分。此外，鉴于物理世界的材料特性，引入微表面理论以描述光和不规则界面的相互作用。微表面BRDF是其需求的具体演绎。
具体请看UE4中的PBR相关笔记。

微表面模型被描述为：

$$
f_x(\bold{v}, \bold{l}) = \frac{1}{|\bold{n}\cdot\bold{v}||\bold{n}\cdot\bold{l}|}\int_\Omega D(\bold{m}, \alpha)G(\bold{v}, \bold{l}, \bold{m}) f_m(\bold{v}, \bold{l}, \bold{m}) (\bold{v} \cdot \bold{m})(\bold{l} \cdot \bold{m}) d\bold{m}
$$

与PBR相同，D用于描述微表面分布，其也被称为NDF或正态分布函数。同理，G模拟了微切面的可见性，也就是几何镜面衰减。其与完整的PBR渲染方程十分相似，只是忽略了菲涅尔因子，此外，不同指出在于该方程在微观上对于半球进行了积分。

### 电介质和导体

为了更好地理解下面显示的一些方程式和行为，我们必须首先清楚地了解金属（导体）表面和非金属（介质）表面之间的区别。
要知道，一部分光会穿透表面(对于万物都是这样)，然后在内部发生散射，再以漫反射的形式离开物体表面。纯金属不会发生次表面反射，这意味着其不存在漫反射部分，电介质则相反，它们既有镜面反射部分，也有漫反射部分。

### 能量守恒

能量守恒是基于物理渲染的良好BRDF的关键要素之一。能量守恒的BRDF表示镜面反射和漫反射能量的总量小于入射能量的总量。
如果没有能量守恒的BRDF，美工人员就必须手动确保表面反射光永远不会比入射光强。

### 镜面反射BRDF

对于镜面项，加入了菲涅尔因子进行建模(期待已久！)，依然是使用熟悉的Cook-Torrance近似模型。

$$
f_r(\bold{v},\bold{l}) = \frac{D(\bold{h}, \alpha) G(\bold{v},\bold{l},\alpha) F(\bold{v}, \bold{h}, \bold{f_0})}{4(\bold{n}\cdot\bold{v})(\bold{n}\cdot\bold{l})}
$$

考虑到实时运算需求，必须对于D,G和F进行近似计算。

### 法线分布函数D

Burley发现，长尾正态分布函数 (NDF) 非常适合真实世界的曲面。Walter中描述的 GGX 分布是一种在高光部分具有长尾落差和短峰值的分布，其公式简单，适合实时实现。它也是现代物理渲染器中的常用模型，相当于Trowbridge-Reitz分布。

$$
D_{GGX}(\bold{h}) = \frac{\alpha^2}{\pi((\bold{n}\cdot\bold{h})^2(\alpha^2-1)+1)^2}
$$

显然，预留了$\alpha$的定义，同时，filament使用半精度浮点数用以改进实现。
半浮点计算$1-(n\cdot h)^2$时存在两个问题，因此这一优化需要对原始方程进行修改。首先，当$(n\cdot h)^2$接近1（亮点）时，这种计算会受到浮点消除的影响。其次，$(n\cdot h)$在1附近没有足够的精度。解决方案设计拉格朗日恒等式：

$$
|a\times b|^2=|a|^2|b|^2−(a\cdot b)^2
$$

n与h皆为单位向量，这说明$1-(n\cdot h)^2 = |n \times h|^2$，通过叉乘避免了这样的误差。

这里给出GLSL的代码实现：

```glsl
#define MEDIUMP_FLT_MAX    65504.0
#define saturateMediump(x) min(x, MEDIUMP_FLT_MAX)

float D_GGX(float roughness, float NoH, const vec3 n, const vec3 h) {
    vec3 NxH = cross(n, h);
    float a = NoH * roughness;
    float k = roughness / (dot(NxH, NxH) + a * a);
    float d = k * k * (1.0 / PI);
    return saturateMediump(d);
}
```

### 几何阴影G

Eric Heitz指出，Smith几何阴影函数(geometric shadowing function)是正确和精确的G术语。Smith公式如下：

$$
G(\bold{v}, \bold{l}, \alpha) = G_1(\bold{l}, \alpha) G_1(\bold{v}, \alpha)
$$

$G_1$又可以遵循几种模式，通常设定为GGX模式：

$$
G_1(\bold{v},\alpha) = G_{GGX}(\bold{v}, \alpha) = \frac{2(\bold{n}\cdot\bold{v})}{\bold{n}\cdot\bold{v} + \sqrt{\alpha^2 + (1 - \alpha^2)(\bold{n}\cdot\bold{v})^2}}
$$

由此，完整的Smith-GGX可以表示为：

$$
G(\bold{v}, \bold{l}, \alpha) =  \frac{2(\bold{n}\cdot\bold{l})}{\bold{n}\cdot\bold{l} + \sqrt{\alpha^2 + (1 - \alpha^2)(\bold{n}\cdot\bold{l})^2}} \frac{2(\bold{n}\cdot\bold{v})}{\bold{n}\cdot\bold{v} + \sqrt{\alpha^2 + (1 - \alpha^2)(\bold{n}\cdot\bold{v})^2}}
$$

Heitz指出，考虑到微面片的高度来关联遮蔽和阴影会得到更准确的结果。他是这样定义与高度相关的Smith函数的：

$$
G(\bold{v}, \bold{l}, \alpha)  = \frac{\chi^+(\bold{v}\cdot\bold{h})\chi^+(\bold{l}\cdot\bold{h})}{1 + \Lambda(\bold{v}) + \Lambda(\bold{l})}
$$

$\chi^+(\alpha)$表示$\alpha > 0$时为1，反之为0

$$
\Lambda(\bold{m}) = \frac{-1 + \sqrt{1 + \alpha^2\tan^2(\theta_m)}}{2}
$$

$\theta_m$代表单位向量m与单位向量n之间的夹角，因此对于向量v来说：

$$
\Lambda(\bold{v}) = \frac{1}{2} (\frac{\sqrt{\alpha^2 + (1 - \alpha^2)(\bold{n}\cdot\bold{v})^2}}{\bold{n}\cdot\bold{v}} - 1)
$$

由此，我们有了：

$$
V(\bold{v}, \bold{l}, \alpha) = \frac{G(\bold{v}, \bold{l}, \alpha)}{4(\bold{n}\cdot\bold{v})(\bold{n}\cdot\bold{l})} = \frac{1/2}{(\bold{n}\cdot\bold{l})\sqrt{(\bold{n}\cdot\bold{v})^2(1-\alpha^2)+\alpha^2} + (\bold{n}\cdot\bold{v})\sqrt{(\bold{n}\cdot\bold{l})^2(1-\alpha^2)+\alpha^2}}
$$

最后，是对于平方根计算的近似：

$$
V(\bold{v}, \bold{l}, \alpha) = \frac{1/2}{(\bold{n}\cdot\bold{l}) [(\bold{n}\cdot\bold{v})(1-\alpha)+\alpha] + (\bold{n}\cdot\bold{v}) [(\bold{n}\cdot\bold{l})(1-\alpha)+\alpha]}
$$

得益于此，GLSL的实现就简单了许多：

```glsl
float V_SmithGGXCorrelated(float NoV, float NoL, float roughness) {
    float a2 = roughness * roughness;
    float GGXV = NoL * sqrt(NoV * NoV * (1.0 - a2) + a2);
    float GGXL = NoV * sqrt(NoL * NoL * (1.0 - a2) + a2);
    return 0.5 / (GGXV + GGXL);
}
```

Hammon基于可以去除平方根的相同观察，提出了相同的近似方法。它是通过将表达式改写成 lerps 来实现的：

$$
V(\bold{v}, \bold{l}, \alpha) = \frac{1/2}{lerp(2(\bold{n}\cdot\bold{l})(\bold{n}\cdot\bold{v}), \bold{n}\cdot\bold{l} + \bold{n}\cdot\bold{v}, \alpha)}
$$

### 菲涅尔因子F

菲涅尔效应对物理材料的外观起着重要作用。这种效应模拟了这样一个事实，即观察者看到的表面反射光量取决于观察角度。

反射光量不仅取决于观察角度，还取决于材料的折射率index of refraction(IOR)。
在正常入射角下（垂直于表面，或 0°角），反射回来的光量记为$f_0$，可以从 IOR 得出，掠射角下的反射光量记为$f_{90}$，对于光滑材料来说接近100%。

菲涅尔项定义了光在两种不同介质界面上的反射和折射情况，或者说是反射能量和透射能量之比。Schlick描述了Cook-Torrance镜面BRDF的菲涅尔项的低成本近似值：

$$
F_{Schlick}(\bold{v},\bold{h}, f_0, f_{90}) = f_0 + (f_{90} - f_0)(1 - \bold{v}\cdot\bold{h})^5
$$

常数$f_0$代表正常入射时的镜面反射率，对于电介质是消色差，对于金属是色差。实际值取决于界面的折射率。该术语的GLSL实现需要使用一个pow，但只需进行几次乘法即可替代。

```glsl
vec3 F_Schlick(float u, vec3 f0, float f90) {
    return f0 + (vec3(f90) - f0) * pow(1.0 - u, 5.0);
}
```

为了方便，可以将$f_{90}$的菲涅尔反射率定义为1。对于该值的进一步定义在后文进行讨论。

### 漫反射 BRDF

在漫反射阶段，我们首先给出漫反射项的定义：

$$
f_d(\bold{v}, \bold{l}) = \frac{c}{\pi}\frac{1}{|\bold{n}\cdot\bold{v}||\bold{n}\cdot\bold{l}|} \int_\Omega D(\bold{m}, \alpha) G(\bold{v}, \bold{l}, \bold{m})(\bold{v}\cdot \bold{m})(\bold{l}\cdot \bold{m})d\bold{m}
$$

可以使用最简单的漫反射染色，filament选择的就是这种方式：

$$
f_d(\bold{v}, \bold{l}) = \frac{c}{\pi}
$$

Disney的模型考略了粗糙度，但是对于移动平台来说可能会带来比较大的运算负荷：

$$
f_d(\bold{v}, \bold{l}) = \frac{c}{\pi}F_{Schlick}(\bold{n},\bold{l}, 1, f_{90}) F_{Schlick}(\bold{n},\bold{v}, 1, f_{90}) 
$$

$$
f_{90} = 0.5 + 2\cdot \alpha cos^2(\theta_d)
$$

对于一块使用完全粗糙介电材料，相比于简单朗伯漫射BRDF，迪斯尼BRDF在掠过角处表现出一些很好的逆反射。

### 标准模型总结

镜面项： 使用的是Cook-Torrance镜面微表面模型，具有GGX正态分布函数、Smith-GGX高度相关能见度函数和Schlick菲涅尔函数。
漫反射项： 使用一个朗伯反射模型

给出一个完整的实现作为参考：

```glsl
float D_GGX(float NoH, float a) {
    float a2 = a * a;
    float f = (NoH * a2 - NoH) * NoH + 1.0;
    return a2 / (PI * f * f);
}

vec3 F_Schlick(float u, vec3 f0) {
    return f0 + (vec3(1.0) - f0) * pow(1.0 - u, 5.0);
}

float V_SmithGGXCorrelated(float NoV, float NoL, float a) {
    float a2 = a * a;
    float GGXL = NoV * sqrt((-NoL * a2 + NoL) * NoL + a2);
    float GGXV = NoL * sqrt((-NoV * a2 + NoV) * NoV + a2);
    return 0.5 / (GGXV + GGXL);
}

float Fd_Lambert() {
    return 1.0 / PI;
}

void BRDF(...) {
    vec3 h = normalize(v + l);

    float NoV = abs(dot(n, v)) + 1e-5;
    float NoL = clamp(dot(n, l), 0.0, 1.0);
    float NoH = clamp(dot(n, h), 0.0, 1.0);
    float LoH = clamp(dot(l, h), 0.0, 1.0);

    // perceptually linear roughness to roughness (see parameterization)
    float roughness = perceptualRoughness * perceptualRoughness;

    float D = D_GGX(NoH, roughness);
    vec3  F = F_Schlick(LoH, f0);
    float V = V_SmithGGXCorrelated(NoV, NoL, roughness);

    // specular BRDF
    vec3 Fr = (D * V) * F;

    // diffuse BRDF
    vec3 Fd = diffuseColor * Fd_Lambert();

    // apply lighting...
}
```

### 提升BRDF

能量守恒是良好BRDF的关键要素之一。针对此定理提出两点问题：

#### 漫反射能量增益

朗伯漫反射不考虑在表面反射的光线。需要明确漫反射与镜面反射带来的反射后光线是否小于等于入射光线。

#### 漫反射能量衰减

Cook-Torrance微表面镜面反射模型试图模拟这样一件事：在粗糙表面的单词光反射，这种近似势必会造成光线的能量损失，对于一个粗糙表面(指存在描述粗糙信息)，可以很轻松理解，会发生更多的多重散射事件，这意味着更多的能量衰减。
随着粗糙程度的增加，能量的衰减是加剧的。

**Multiple-Scattering Microfacet BSDFs with the Smith Model**提出了多散射微表面，但是filament仅仅给出了多散射BRDF的随机评估，因此这种解决方案并不适用于实时渲染，**Revisiting Physically Based Shading at Imageworks**给出了一种新的方案，通过增加一个能量补偿项作为额外的BRDF波瓣：

$$
f_{ms}(\bold{l}, \bold{v}) = \frac{(1 - E(\bold{l}))(1 - E(\bold{v}))F^2_{avg} E_{avg}}{\pi(1 - E_{avg})(1 - F_{avg}(1 - E_{avg}))}
$$

E是对于镜面反射BRDF的方向性albedo，且假定$f_0 = 1$:

$$
E(\bold{l}) = \int_\Omega f(\bold{l}, \bold{v})(\bold{n}, \bold{v})d \bold{v}
$$

$E_{avg}$是E的余弦加权平均值：

$$
E_{avg} = 2\int^1_0E(\mu)\mu d\mu
$$

同样，$F_{avg}$是菲涅尔项的余弦加权平均值：

$$
F_{avg} = 2\int^1_0F(\mu)\mu d\mu
$$

E和$E_{avg}$都可以预先计算并存储在查找表中，而$F_{avg}$在使用Schlick近似法时可以大大简化。

### 参数化

filament使用的材质参数系统是基于Disney的渲染方案进行参数的优化。
下面是各项参数：

**BaseColor**: 非金属表面的扩散反照率和金属表面的镜面颜色。
**Metallic**: 表面看起来是介质(0.0)还是导体(1.0),通常用作二进制值(0或1)。
**Roughness**: 表面的光滑度(0.0)或粗糙度(1.0),光滑表面带来更加清晰的反射。
**Reflectance**: 介质表面法线入射时的菲涅尔反射率，这取代了明确的折射率。
**Emissive**：额外的漫反射反照率，用于模拟发射性表面。该参数主要用于HDR流水线中的Bloom pass。
**Ambient occlusion**: 定义表面点可以获得多少环境光。用于描述每个像素的阴影系数介于0.0和1.0之间。

参数的定义并非绝对的，可能会进行一定的线性/非线性变换，例如，BaseColor可以用sRGB空间表示，并在发送给着色器之前转换为线性空间。用0到255之间的灰度值（从黑到白）来表示metallic、roughness和reflectance对开发者来说也很有用。

### 对参数的重新映射

基于主流的标准材质模型，filament对于baseColor,roughness和reflectance进行了重新映射。

#### **BaseColor** remapping

材料的**BaseColor**受其 "金属性 "的影响。电介质具有非彩色镜面反射，但其**BaseColor**保留为漫反射色。而导体则将**BaseColor**作为镜面反射色，没有漫反射成分。
照明方程必须使用漫反射颜色和$f_0$而不是**BaseColor**。
**漫反射**颜色可以很容易地从**BaseColor**中计算出来:

```glsl
vec3 diffuseColor = (1.0 - metallic) * baseColor.rgb;
```

#### **Reflectance** remapping

##### 电介质

菲涅尔项依赖于$f_0$，即正常入射角下的镜面反射率，对于电介质是消色差的:

$$
f_0 = 0.16 \cdot reflectance^2
$$

这样可以保证将$f_0$映射到能代表普通介质表面(4%反射率)到宝石(8~16%反射率)，在输入**Reflectance**为0.5时提供了4%的菲涅尔反射率。总体上来说保证了一条斜率逐渐升高的平滑曲线。

如果折射率(Index of Refraction)已知, 例如，空气-水界面的折射率为1.33，则菲涅尔反射率可按下式计算：

$$
f_0 = \frac{(n_{ior} - 1)^2}{(n_{ior} + 1)^2}
$$

也顺便给出反推公式，仅基于上式：

$$
n_{ior} = \frac{2}{1 - \sqrt{f_0}} - 1
$$

此外，在掠入射角的情况下，所有材料的菲涅尔反射率都被定义为100%。

##### 导体

金属表面的镜面反射是有色差的：

$$
f_0 = baseColor \cdot metallic
$$

##### 兼具电介质和导体

```glsl
vec3 f0 = 0.16 * reflectance * reflectance * (1.0 - metallic) + baseColor * metallic;
```

#### **Roughness** remapping

定义一个感知粗糙度量(perceptualRoughness), 描述开发者定义的粗糙度，并将其重新映射到感知线性范围。

$$
\alpha = perceptualRoughness^2
$$

通过这种处理，保证物体在低Roughness的情况下，表现出变化更加缓慢的光滑特性。相较于原始Roughness，达到一个相同的粗糙表面效果所需要的值更高。
如果没有这种重映射，闪亮的金属表面就只能局限在0.0和0.05之间的极小范围内。
Disney出于类似的动机进行了重映射，filament选择简单的平方同时兼顾了低廉的计算成本。

另外一个问题是，有限的浮点运算也许会导致些许问题，在移动端GPU上，粗糙度一般是以半浮点数(16bit)的形式实现的。为了保证诸如这样的式子$\frac{1}{perceptualRoughness^4}$不会使分母过小被表示为0，使得perceptualRoughness的最小值不能低于$2^{-14}$或者说$6.1 \times 10^{-5}$，因此省去计算过程，结论上roughness不能够小于$6.274 \times 10^{-5}$。

最后，在粗糙度(十分)接近0时会产生几乎不可见的高光，粗糙度应该处于一个安全的范围，保证反射的lobe足够宽到可以被观察到(我猜想是出于这个需求)，另一个需求是可以纠正粗糙度过低造成的镜面反差。

### 混合和分层

材料的混合和分层实际上是对材料模型的各种参数进行插值。比如在闪亮的金属铬和光滑的塑料之间进行插值，以模拟出更多的材料视觉效果。

### 总结：如何创建真实材质

#### 所有材料

**Base color**：除micro-occlusion外，应不提供照明信息

**Metallic**：应当接近于二进制，理论上纯导体应当为1，纯电介质应当为0。

#### 非金属材料

**Base color**：代表反射颜色，应为50-240（严格范围）或30-240（容许范围）范围内的sRGB值。

**Metallic**：为0或尽可能接近0.

**Reflectance**：如果找不到合适的值，应将其设置为127sRGB（0.5线性，4%反射率）。不要使用低于 90 sRGB（0.35线性，2%反射率）的值。

#### 金属材料

**Base color**：代表镜面颜色和反射率。使用亮度值为67%到100%（170-255 sRGB）。考虑到非金属成分，氧化或脏污金属的亮度应低于清洁金属。

**Metallic**：为1或尽可能接近1.

**Reflectance**：会被忽略（根据**Base color**计算方式）。

### 涂层模型

多层材料相当常见，特别是在标准层上有一层薄薄的半透明层的材料。这类材料在现实世界中的例子包括汽车漆、苏打罐、漆木、丙烯酸等。

透明涂层可以作为标准材料模型的扩展进行模拟，相当于在原有的反射模型表面上再考虑一层进行镜面BRDF计算，为了简化实现，将透明图层始终定义为各向同性。而底层理论上可以是任何一种材料。

此外需要考虑能量的进出损失，但是简化了涂层和底部材料之间的反射和折射现象。
透明涂层将采用与标准模型相同的Cook-Torrance microfacet BRDF进行建模。由于透明涂层始终是各向同性和介电的，粗糙度值很低。因而选择更加便宜的DFG项，并保证尽可能不会影响视觉质量。

依据**A Microfacet Based Coupled Specular-Matte BRDF Model with Importance Sampling**给出了代替Smith-GGX的可见性项：

$$
V(l, h) = \frac{1}{4(l\cdot h)^2}
$$

这定义并不符合物理，仅仅以其简单性符合于实时渲染的场景。

进一步地，镜面BRDF的菲涅尔项需要$f_0$，即正常入射角下的镜面反射率。这个参数可以通过界面的折射率计算出来。我们假设透明涂层由聚氨酯或类似材料制成，聚氨酯是涂料和清漆中常用的化合物。空气-聚氨酯界面的折射率为 1.5，由此可以推算出$f_0$：

$$
f_0(1.5) = \frac{(1.5 - 1)^2}{(1.5 + 1)^2} = 0.04
$$

这相当于4%的菲涅尔反射率，我们知道这与普通电介质材料有关。

我们知道反射模型由镜面反射($f_r$)和漫反射($f_d$)组成，因而作进一步的整合：

$$
f(v, l) = f_d(v, l)(1 - F_c) + f_r(v, l)(1 - F_c) + f_c(v, l)
$$

使用$F_c$定义透明图层BRDF的菲涅尔项。

#### 透明图层的参数

除了上文的BRDF提供的参数，还包含额外两个参数：

**ClearCoat**: 透明涂层的强度。定义为0和1之间的标量。
**ClearCoatRoughness**: 透明涂层的平滑度或粗糙度。定义为0和1之间的标量。

简单给出GLSL中相关实现：

```glsl
void BRDF(...) {
    // compute Fd and Fr from standard model

    // remapping and linearization of clear coat roughness
    clearCoatPerceptualRoughness = clamp(clearCoatPerceptualRoughness, 0.089, 1.0);
    clearCoatRoughness = clearCoatPerceptualRoughness * clearCoatPerceptualRoughness;

    // clear coat BRDF
    float  Dc = D_GGX(clearCoatRoughness, NoH);
    float  Vc = V_Kelemen(clearCoatRoughness, LoH);
    float  Fc = F_Schlick(0.04, LoH) * clearCoat; // clear coat strength
    float Frc = (Dc * Vc) * Fc;

    // account for energy loss in the base layer
    return color * ((Fd + Fr * (1.0 - Fc)) * (1.0 - Fc) + Frc);
}
```

最后，透明涂层的存在意味着我们应该重新计算$f_0$，因为它通常是基于空气-材料界面。因此，底层需要根据透明涂层-材料界面来计算$f_0$。
这一步是通过整合上文的求取$f_0$和$IOR$合并进行的：

$$
f_{0_{base}} = \frac{(1 - 5\sqrt{f_0})^2}{(5 - \sqrt{f_0})^2}
$$

另一方面，重新定义映射粗糙度也应当纳入考虑。

### 各向异性模型

之前描述的标准材料模型只能用于各向同性的表面，也就是在所有方向上属性相同的表面。但对于各向异性表面的考虑不可谓不重要。

#### 各向异性镜面BRDF反射

通过修改前文的各向同性BRDF以获得各向异性的GGX NDF函数;

$$
D_{aniso}(h, \alpha) = \frac{1}{\pi\alpha_t \alpha_b}\frac{1}{((\frac{t\cdot h}{\alpha_t})^2 + (\frac{b\cdot h}{\alpha_b})^2 + (n\cdot h)^2)^2}
$$

这种NDF依赖于两个补充粗糙度项，沿位切线方向的粗糙度$\alpha_b$和沿切线方向的粗糙度$\alpha_t$,于**Crafting a Next-Gen Material Pipeline for The Order: 1886**提供了一种描述两个粗糙度值之间的关系的各向异性参数：

$$
\alpha_t = \alpha
$$

$$
\alpha_b = lerp(0, \alpha, 1 - anisotropy)
$$

Disney给出了一个成本略高的方案：

$$
\alpha_t = \frac{\alpha}{\sqrt{1 - 0.9 \times anistropy}}
$$

$$
\alpha_b = \frac{1}{\alpha\sqrt{1 - 0.9 \times anistropy}}
$$

为了能够展现更直观的高光，filament选择了**Revisiting Physically Based Shading at Imageworks**的方案：

$$
\alpha_t = \alpha \times (1 + anistropy)
$$

$$
\alpha_b = \alpha \times (1 - anistropy)
$$

除法线方向外，该 NDF 还需要切线和位切线方向。由于法线映射已经需要这些方向，因此提供它们可能不成问题。
具体的实现为：

```glsl
float at = max(roughness * (1.0 + anisotropy), 0.001);
float ab = max(roughness * (1.0 - anisotropy), 0.001);

float D_GGX_Anisotropic(float NoH, const vec3 h,
        const vec3 t, const vec3 b, float at, float ab) {
    float ToH = dot(t, h);
    float BoH = dot(b, h);
    float a2 = at * ab;
    highp vec3 v = vec3(ab * ToH, at * BoH, a2 * NoH);
    highp float v2 = dot(v, v);
    float w2 = a2 / v2;
    return a2 * w2 * w2 * (1.0 / PI);
}
```

**Understanding the Masking-Shadowing Function in Microfacet-Based BRDFs**给出了针对各向异性的掩码-遮盖函数(masking-shadowing function)，用于匹配高度相关的GGX分布，使用可见度函数优化掩码-遮盖项

$$
G(\bold{v}, \bold{l}, \bold{h}, \alpha)  = \frac{\chi^+(\bold{v}\cdot\bold{h})\chi^+(\bold{l}\cdot\bold{h})}{1 + \Lambda(\bold{v}) + \Lambda(\bold{l})}
$$

$$
\Lambda(\bold{m}) = \frac{-1 + \sqrt{1 + \alpha_0^2\tan^2(\theta_m)}}{2}
$$

令：

$$
\alpha_0 = \sqrt{cos^2(\phi_0)\alpha^2_x + sin^2(\phi_0)\alpha^2_y}
$$

由此得到可见性函数:

$$
V_aniso(\bold{n} \cdot \bold{l}, \bold{n} \cdot \bold{v}, \alpha) = \frac{1}{2((\bold{n} \cdot \bold{l})\hat{\Lambda}_v + (\bold{n} \cdot \bold{v})\hat{\Lambda}_l)}
$$

$$
\hat{\Lambda}_v = \sqrt{\alpha_t^2(\bold{t} \cdot \bold{v})^2 + \alpha_b^2(\bold{b} \cdot \bold{v})^2 + (\bold{n} \cdot \bold{v})^2}
$$

$$
\hat{\Lambda}_l = \sqrt{\alpha_t^2(\bold{t} \cdot \bold{l})^2 + \alpha_b^2(\bold{b} \cdot \bold{l})^2 + (\bold{n} \cdot \bold{l})^2}
$$

$\hat{\Lambda}_l$对于每种光线都是相同的。具体实现：

```glsl
float at = max(roughness * (1.0 + anisotropy), 0.001);
float ab = max(roughness * (1.0 - anisotropy), 0.001);

float V_SmithGGXCorrelated_Anisotropic(float at, float ab, float ToV, float BoV,
        float ToL, float BoL, float NoV, float NoL) {
    float lambdaV = NoL * length(vec3(at * ToV, ab * BoV, NoV));
    float lambdaL = NoV * length(vec3(at * ToL, ab * BoL, NoL));
    float v = 0.5 / (lambdaV + lambdaL);
    return saturateMediump(v);
}
```

#### 各向异性参数化

针对各向异性的材料提供一个额外的参数$Anisotropy$，其位于[-1, 1]之间。
负值将使各向异性与位切线方向而非正切方向保持一致。

### 布料反射模型

衣服和织物通常是由松散的线连接而成，会吸收和散射入射光。前面介绍的微面 BRDF 无法很好地再现布料的特性，因为它们的基本假设是表面由随机凹槽组成，这些凹槽的行为就像完美的镜子。与硬质表面相比，布料的特点是镜面叶较软，落差较大，并且存在由前向/后向散射引起的模糊光。有些织物还会呈现双色镜面色彩（例如天鹅绒）。
天鹅绒是一个有趣的使用案例。由于前向和后向散射，这种布料表现出强烈的边缘光。这些散射事件是由直立在织物表面的纤维造成的。当入射光来自与视线方向相反的方向时，纤维会向前散射光线。同样，当入射光线与视线方向相同时，纤维会向后散射光线。

#### 布料镜面反射

**Distribution-based BRDFs**提出，分布项对BRDF的贡献最大，而对于Ashikhmin and Premoze所提出的的天鹅绒分布而言，阴影/遮蔽项并非必要。分布项本身是一个倒高斯分布。这有助于实现模糊照明（前向和后向散射），同时添加偏移量来模拟前向镜面反射。所谓的天鹅绒NDF定义如下：

$$
D_{velvet}(v, h, \alpha) = c_{norm}(1 + 4 e^{(\frac{-cot^2\theta_h}{\alpha^2})})
$$

**Crafting a Next-Gen Material Pipeline for The Order: 1886**提出了该NDF的规范化版本：

$$
D_{velvet}(v, h, \alpha) = \frac{1}{\pi (1 + 4\alpha^2)}(1 + 4\frac{e^{(\frac{-cot^2\theta_h}{\alpha^2})}}{sin^4 \theta_h})
$$

并继续采用其全镜面BRDF方案:

$$
f_r(v, h, \alpha) = \frac{D_{velvet}(v, h, \alpha)}{4(\bold{n}\cdot\bold{l} + \bold{n}\cdot\bold{v} - (\bold{n}\cdot\bold{l})(\bold{n}\cdot\bold{v}))}
$$

该方案并不包含菲涅尔项。
Ashikhmin's velvet NDF具体的实现：

```glsl
float D_Ashikhmin(float roughness, float NoH) {
    // Ashikhmin 2007, "Distribution-based BRDFs"
  float a2 = roughness * roughness;
  float cos2h = NoH * NoH;
  float sin2h = max(1.0 - cos2h, 0.0078125); // 2^(-14/2), so sin2h^2 > 0 in fp16
  float sin4h = sin2h * sin2h;
  float cot2 = -cos2h / (a2 * sin2h);
  1.0 / (PI * (4.0 * a2 + 1.0) * sin4h) * (4.0 * exp(cot2) + sin4h);
}
```

**Production Friendly Microfacet Sheen BRDF**基于指数化正弦曲线而不是倒高斯曲线。这种 NDF 有几个吸引人的原因：其参数化感觉更自然、更直观，外观更柔和，实现也更加简单：

$$
D(m) = \frac{(2 + \frac{1}{\alpha}sin(\theta)^\frac{1}{\alpha})}{2\pi}
$$

```glsl
float D_Charlie(float roughness, float NoH) {
    // Estevez and Kulla 2017, "Production Friendly Microfacet Sheen BRDF"
    float invAlpha  = 1.0 / roughness;
    float cos2h = NoH * NoH;
    float sin2h = max(1.0 - cos2h, 0.0078125); // 2^(-14/2), so sin2h^2 > 0 in fp16
    return (2.0 + invAlpha) * pow(sin2h, invAlpha * 0.5) / (2.0 * PI);
}
```

filament采取了其NDF并忽略了阴影项。

为了更好地控制布料的外观，并让开发者能够重新创建双色镜面材料，filament引入了直接修改镜面反射率的功能。

#### 布料漫反射

针对于布料的漫反射模型依然依赖于朗伯漫反射模型，目的是为了提供并非基于物理而是为了模拟织物的散射、部分吸收和再发射的效果。
首先是不包含可选次表面散射的漫反射项:

$$
f_d(v, h) = \frac{c_{diff}}{\pi}(1 - F(v, h))
$$

$F(v, h)$是布料镜面BRDF的菲涅尔项，实际开发中依然会选择被省去，这是权衡优化和效果之后的选择。

次表层散射采用能量守恒形式的包裹式漫射照明技术：

$$
f_d(v, h) = \frac{c_{diff}}{\pi}(1 - F(v,h)) \langle \frac{\bold{n} \cdot \bold{l} + w}{(\bold{l} + w)^2} \rangle \langle c_{subsurface} + \bold{n} \cdot \bold{l}\rangle
$$

其中，$w$是一个介于0和1之间的值，它定义了漫射光线环绕明暗分界的程度。为了避免引入另一个参数，将$w$固定为0.5。需要注意的是，在使用环绕漫射光时，漫射项不能乘以$\bold{n} \cdot \bold{l}$

织物BRDF的完整实现，包括光泽颜色和可选的次表面散射：

```glsl
// specular BRDF
float D = distributionCloth(roughness, NoH);
float V = visibilityCloth(NoV, NoL);
vec3  F = sheenColor;
vec3 Fr = (D * V) * F;

// diffuse BRDF
float diffuse = diffuse(roughness, NoV, NoL, LoH);
#if defined(MATERIAL_HAS_SUBSURFACE_COLOR)
// energy conservative wrap diffuse
diffuse *= saturate((dot(n, light.l) + 0.5) / 2.25);
#endif
vec3 Fd = diffuse * pixel.diffuseColor;

#if defined(MATERIAL_HAS_SUBSURFACE_COLOR)
// cheap subsurface scatter
Fd *= saturate(subsurfaceColor + NoL);
vec3 color = Fd + Fr * NoL;
color *= (lightIntensity * lightAttenuation) * lightColor;
#else
vec3 color = Fd + Fr;
color *= (lightIntensity * lightAttenuation * NoL) * lightColor;
#endif
```

#### 布料反射模型参数

同样，我们需要为该反射模型提供些许参数：

**SheenColor**: 镜面色调，用于创建双色镜面织物（默认值为 0.04，以匹配标准反射率）
**SubsurfaceColor**: 材料散射和吸收后的漫射色调

要创建类似天鹅绒的材料，可将基色设置为黑色（或深色）。
色度信息应设置为光泽色。要制作更常见的织物，如牛仔布、棉布等，可将基色用于色度，并使用默认的光泽色或将光泽色设置为基色的亮度。