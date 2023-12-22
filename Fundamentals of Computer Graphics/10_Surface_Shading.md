# chapter-10 Surface Shading

## 漫反射

朗博体(Lambertian objects)：完全没有光泽，颜色不会随着视点的变化而变化。

### 朗博着色模型

朗博余弦定理：表面的颜色与表面法线和光源方向之间的角度的余弦成比例

$$
c = c_rc_lmax(\mathbf{n}·\mathbf{l}, 0)
$$

## 环境光

直观地说，可以将环境颜色$c_a$视为场景中所有曲面的平均颜色。如果要确保计算的RGB颜色保持在[0,1]3的范围内，则$c_a+c_l \le (1,1,1)$

## Phong渲染

前面已经说过了，Phong所添加的光照是基于视点变换所产生变换的镜面反射光线。

$$
c = c_l max(0, \mathbf{e}·\mathbf{r})^p
$$

也就是：

$$
h = \frac{\mathbf{e} + \mathbf{l}}{||\mathbf{e} + \mathbf{l}||}
$$

$$
c = c_l max(0, \mathbf{h}·\mathbf{n})^p
$$

p就是反映高亮强度的指数，越高光亮越集中。

## 艺术着色


