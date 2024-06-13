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