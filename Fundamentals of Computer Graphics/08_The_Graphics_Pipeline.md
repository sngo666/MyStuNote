# chapter-08 The Graphics Pipeline

## 光栅化

### 画线

使用隐式方程画线最常见的方法是中点算法。中点算法最终绘制的线与Bresenham算法相同，但它更简单。
当需要在$(x_0, y_0)$和$(x_1, y_1)$之间画一条直线，问题在于将哪些像素点串联起来形成一条最直线的直线。

斜率：

$$
m = \frac{y1- y0}{x1- x0}
$$

斜率存在四种情况，在0到1之间，1到正无穷之间，以及这两种情况的相反数。
对于这几种情况的讨论，只需讨论一种情况就能很轻松推导出其他几种情况，在实际 实现中，可以对调x和y的坐标将需要考虑的情况数量缩减到两种。

当m在0到1之间，说明x移动的速度快于y，每当x向右移动一个像素点，y不变的可能性更大(或者说这种情况的比重更大)。
> 我们需要保证的是，在两个端点之间，每一列都最好只有一个像素点被设置颜色，这样画出来的直线能够保证稳定的细度。

从设计来说，其实就是在横向的角度，从端点的一侧到另一侧判断哪个y方向上的像素点应该被设置。
在每个判断中，判断直线穿过$(x+1, y)$也就是最新的已标记像素点右侧像素点的中心位置的上方还是下方，在中心上方则绘制顶部像素，否则绘制底部像素。

具体表现为判断$f (x_0 + 1, y_0 + 0.5) < 0$，说明这条假定的线在需要绘制的线$f (x_0, y_0)$下方，则需要绘制在右上方像素格，其中：

$$
f(x+1, y) = f(x, y) + (y_0 - y_1)
$$

根据：

$$
f(x, y) = (y_0-y_1)x + (x_1 - x_0)y + C
$$
将两个点代入，得到：

$$
C = x_0y_1 - x_1y_0
$$

当我们不关心C，可以发现，所有A和B相同的直线，都是无数个相互平行的直线，而C决定了它们之间的距离。

## 三角形的光栅化

Gouraud插值：
对于组成三角形的三个点$p_0=(x_0, y_0),p_1=(x_1, y_1),p_2=(x_2, y_2)$. 三角形内部的点可以表示为$(α,β,γ)$，三个系数之和为0，且都为正数:

$$
c = αc_0 + βc_1 + γc_2
$$

其中$c_0,c_1,c_2$代表三个点被设定的颜色(纹理值)，c代表内部一点的颜色值(纹理值)
这种插值用于处理三角形内部的绘制，对于两片三角形边界上的点，后面会有方法处理。

在先前讨论关于插值的三个系数计算中，给出了如下的公式：

$$
β = \frac{f_{ac}(x,y)}{f_{ac}(x_b,y_b)}
$$

简单证明：

对于所有的C，当选取的点组合{($x_0, y_0$), ($x_1, y_1$)}相同时组成的直线集合，都是一组互相平行的直线，而C决定了其与C=0时的直线的垂直距离，因此当我们需要用$f_{ac}(x_b,y_b)$作为分母以表示一个极限的系数，因为我们需要的插值都在三角形内，而经过点b的直线到经过ac两点的距离表示的系数应当为1，从而保证三角形内经过任意一点并平行于ac边的直线能够表示的距离系数在$(0,1)$。

理论依据：

插值的三个系数是通过面积的对比计算出来的，即从插值点出发，到三个顶点所切割的三个三角形，以ac边为底的三角形占整个三角形面积的比重，就是β，而计算上，两个三角形的底边相同，因此比值计算就会变成了插值点和点b到ac边的垂直距离之比。

因为我们计算绘制像素时是按x或者y方向上进行固定的方向绘制，有时候可以省去计算插值系数的过程，沿用规律直接计算插值。

### 三角形边缘的光栅化

在两个三角形之间边缘的像素，必须定义其颜色如何计算。
最坏的选择是不着色，同时考虑两边三角形而着色也未必是个好的选择，最好是选择其中一个三角形，大部分情况下，选择哪个其实并不重要。

### 切割

剪裁是图形中的一种常见操作，每当一个几何图元“剪切”另一个几何实体时都需要进行剪裁。

实现剪裁的两种最常见的方法是

1. 在世界坐标中，使用约束截断的观察棱锥体的六个平面。
2. 在齐次分割之前的4D变换空间中。

方法1非常的简单

### 通过平面切割

假如平面上存在点p,q，对于该平面的任意法线向量n，有：

$$
f(p) = n·(p-q) = 0
$$

变换成形式：

$$
f(p) = n·p + D = 0
$$

现在回到切割的问题上来，对于点a和b相连的线段，存在一点p，若有f(p)<0为平面内，f(p)>0为平面外，如果平面将直线分开，那么就可以通过代入参数直线的方程来解出交点：

$$
p = a + t(b-a)
$$

$$
n·(a + t(b-a)) + D = 0
$$

从而很容易求出t。

## 光栅化前后的操作

在对基本体进行光栅化之前，定义该基本体的顶点必须位于屏幕坐标中，并且必须知道应该在基本体上插值的颜色或其他属性。准备这些数据是管道的顶点处理阶段的工作。在这个阶段，传入顶点通过建模、查看和投影变换进行变换，将它们从原始坐标映射到屏幕空间。

### 简单2D绘制

在混合阶段，每个片段的颜色只是覆盖前一个片段的值。应用程序直接以像素坐标提供基元，光栅化器完成所有工作。

### 最小的3D Pipeline

为了获得正确的遮挡关系，必须要按顺序绘制基本体，但是这种方法可能无法正确处理彼此相交的三角形，此外按照深度对于基元进行排序会很慢，对于大型场景来说，这会是致命的问题。

### 使用深度缓冲算法

在实际使用中，画家算法往往很少被使用(指如同画家作画般，基于深度一层一层的渲染和覆盖)，z缓冲算法是更加简单有效的方法，在每个像素航，跟踪到目前为止最近曲面的距离，并丢弃距离更远的碎片。在实际存储的过程中，除了每个像素点位上的色值，还会存储一个额外的值用于存储渲染来源的距离。

* 精度问题

一般来说，对于深度的存储牵扯到一个缩放问题，我们的数据结构一般不允许一个过大或者说上限很高的深度值被存储，特别是对于每个像素都要存储这个状况来说，比较合理的做法是将最远的深度(f)和近点(n)之间的距离进行一个整体的缩放，这就会造成一定程度的精度损失，甚至影响深度缓冲的最终结果。

总而言之，距离上的缩放，势必会导致某些过小的差值造成的误差。

### 逐顶点着色

也称为Gouraud着色，处理着色计算的一种方法是在顶点阶段执行这些计算。该应用程序在顶点处提供法线向量，并且单独提供灯光的位置和颜色。对于每个顶点，将根据摄影机、灯光和顶点的位置计算到查看器的方向和到每个灯光的方向。计算所需的着色方程式以计算颜色，然后将其作为顶点颜色传递给光栅化器。

逐顶点着色的缺点是，它无法在着色中生成比用于绘制曲面的基本体更小的任何细节，因为它只为每个顶点计算一次着色，而从不在顶点之间计算。

### 逐片元着色

在逐片段着色中，会对相同的着色方程式进行求值，但会使用插值向量对每个片段进行求值，而不是使用应用程序中的向量对每个顶点进行求值。在逐片段着色中，着色所需的几何信息作为属性通过光栅化器，因此顶点阶段必须与片段阶段协调，以适当地准备数据。

### 纹理贴图

第十一章详细讨论

### 着色频率

放置着色计算的决定取决于颜色变化的速度——计算细节的比例。

具有大规模特征的着色（如曲面上的漫反射着色）可以很少进行评估，然后进行插值：可以使用较低的着色频率进行计算。
生成小规模特征（如清晰高光或详细纹理）的着色需要以较高的着色频率进行评估。
对于需要在图像中看起来清晰的细节，着色频率需要为每个像素至少一个着色采样。

## 简单抗锯齿(Simple Antialiasing)

文献来源：《The use of grayscale for improved raster display of vectors and characters》

## 调整渲染顺序以提高效率

识别和扔不可见的几何图形，以节省处理所花费的时间，这被称为剔除，主要分为以下几点：

去除视图体积外的几何形状。
去除可能在视图体积内但被遮挡或遮挡的几何，其他几何形状更接近相机。
去除远离相机的基元。

参考《Real-Time Rendering》Tomas Akenine-Möller
