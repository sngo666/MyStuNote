# chapter-06 Transformation Matrices

## 二维坐标系下基础变换

### Scaling

没什么特别需要说明的，二维变换：

$$
scale(s_x, s_y) = \begin{bmatrix} 
s_x & 0\\
0 & s_y
\end{bmatrix}
$$

### Shearing

Shearing是把东西推到一边的东西，产生一副你把手推过的卡片；最下面的牌保持不动，牌越靠近牌组的顶部移动得越多。

$$
shear_x(s_x, s_y) = \begin{bmatrix} 
1 & s\\
0 & 1
\end{bmatrix}
$$
$$
shear_y(s_x, s_y) = \begin{bmatrix} 
1 & 0\\
s & 1
\end{bmatrix}
$$

### Rotation

逆时针旋转：

$$
rotate(φ) = \begin{bmatrix}
cos φ & − sin φ\\
sin φ & cos φ
\end{bmatrix}
$$

### Reflection

反射矩阵$reflect(φ)$代表以$\frac{φ}{2}$为轴进行翻转。

$$
reflect(φ) = \begin{bmatrix}
cos φ & sin φ\\
sin φ & -cos φ
\end{bmatrix}
$$

$$
reflect_y = \begin{bmatrix}
-1 & 0\\
0 & 1
\end{bmatrix}
$$
$$
reflect_x = \begin{bmatrix}
1 & 0\\
0 & -1
\end{bmatrix}
$$

### 变换的分解

非均匀缩放只能沿着坐标轴进行。提出一种按特定方向进行缩放的整合：

$$
rotate(−45^◦) scale(1.5, 1) rotate(45^◦).
$$

现在再看一个对称矩阵：

$$
A = RSR^T
$$

将R看作一个旋转矩阵，S是一个对角矩阵，那么也可以被看作是一个缩放矩阵
这告诉我们对称矩阵意味着什么：对称矩阵只是缩放操作，而不是潜在的非均匀和非轴对齐操作。

#### 奇异值分解

先前介绍过奇异值分解，代入变换矩阵后进一步讨论：

$$
A = USV^T
$$

替换单个旋转R的两个正交矩阵称为U和V，它们的列分别称为$u_i$（左奇异向量）和$v_i$（右奇异向量）。在这种情况下，S的对角项被称为奇异值，而不是特征值。

SVD总是产生一个具有所有正项的对角矩阵，但矩阵U和V不能保证是旋转的——它们也可能包括反射。

SVD存在的一个直接结果是，我们所看到的所有2D变换矩阵都可以由旋转矩阵和缩放矩阵构成。

#### Paeth旋转分解

这种特殊的变换对于栅格旋转很有用:

$$
\begin{bmatrix}
cos φ & − sin φ\\
sin φ & cos φ
\end{bmatrix} = \begin{bmatrix}
1 & \frac{cos φ - 1}{sin φ} \\
1 & 1
\end{bmatrix}\begin{bmatrix}
1 & 0 \\
sin φ & 1
\end{bmatrix}\begin{bmatrix}
1 & \frac{cos φ - 1}{sin φ} \\
0 & 1
\end{bmatrix}
$$

将SHearing矩阵中的平移项(如$\frac{cos φ - 1}{sin φ}$)四舍五入到最接近的整数，这相当于取图像中的每一行并将其向侧面移动一些量—每行移动不同的量。因为它在一行中是相同的位移，这允许我们在结果图像中没有间隙的情况下旋转。类似的作用也适用于垂直剪切。因此，我们可以很容易地实现一个简单的光栅旋转。

## 3D线性变换

### 三板斧

$$
scale(s_x, s_y , s_z ) = \begin{bmatrix}
s_x & 0 & 0 \\ 0 & s_y & 0 \\ 0 & 0 & s_z
\end{bmatrix}
$$

$$
rotate-z(φ) = \begin{bmatrix}
cos φ & − sin φ & 0 \\ sin φ & cos φ & 0 \\ 0 & 0 & 1
\end{bmatrix}
$$

$$
rotate-x(φ) = \begin{bmatrix}
1 & 0 & 0 \\ 0 & cos φ & − sin φ \\0 & sin φ & cos φ
\end{bmatrix}
$$

$$
rotate-y(φ) = \begin{bmatrix}
cos φ & 0 & sin φ \\0 & 1 & 0 \\− sin φ & 0 & cos φ
\end{bmatrix}
$$

$$
shear-x(d_y , d_z) = \begin{bmatrix}
1 & d_y & d_z \\ 0 & 1 & 0 \\ 0 & 0 & 1
\end{bmatrix}
$$

任何3D变换矩阵都可以使用SVD分解为旋转、缩放和另一个旋转。任何对称的三维矩阵都有一个特征值分解成旋转、比例和逆旋转。最后，三维旋转可以分解为三维Shearing矩阵的乘积。

### 任意旋转矩阵

和在2D中一样，3D旋转是正交矩阵，3D旋转是正交矩阵。在几何上，这意味着矩阵的三行是三个相互正交的单位向量的笛卡尔坐标。列是三个可能不同的相互正交的单位向量：

$$
R_{uvw} = \begin{bmatrix}
x_u & y_u & z_u \\ x_v & y_v & z_v \\ x_w & y_w & z_w
\end{bmatrix}
$$

从几何上讲，这意味着矩阵的三行是三个相互正交的单位向量的笛卡尔坐标。

假设:

$$
u = x_u\mathbf{x} + y_u\mathbf{y} + z_u\mathbf{z}
$$

对于$\mathbf{v}$和$\mathbf{w}$同理
不难得到

$$
R_{uvw}u = \begin{bmatrix} 1 \\ 0 \\ 0 \end{bmatrix} = \mathbf{x}
$$

$$
R_{uvw}v = \mathbf{y},\\
R_{uvw} w = \mathbf{z}
$$

$R_{uvw}$通过旋转将基础uvw带到相应的笛卡尔轴。
对于变换矩阵，代数逆也是几何逆。因此，如果$R_{uvw}$将u带到$\mathbf{x}$，那么$R_{uvw}$将x带到u。我们可以确认，v和$\mathbf{y}$也是如此。

我们总是可以从正交基创造旋转矩阵。

如果我们希望绕任意向量a旋转，我们可以形成w=a的正交基，将该基旋转到规范基xyz，绕z轴旋转，然后将规范基旋转回uvw基：

$$
R_{uvw}^TR_{(φ)}R_{uvw}
$$

首先是需要构建三个正交基组成的轴，以旋转轴计算其存在的两个轴，然后组成一个正交基。

### 变换法线向量

有时候我们可能需要传递一个面的法向量来进行一些计算，例如光线的反射等。当面经过变换M后，面的法向量相应的，可能不会再垂直于这个面。

$$
n^Tt = 0
$$

假设一个变换矩阵N，将法线向量n带到垂直于变换后的表面的向量。
假设所需向量$t_M = Mt$以及变换后的法线向量$n_N = Nn$

$$
n^Tt = n^TIt = n^TM^{-1}Mt = (n^TM^{-1})t_M = 0
$$

易得：

$$
n_N^T = n^TM^{-1}
$$

即：

$$
n_N = (m^{-1})^Tn
$$

## 转换与仿射变换

通过添加一个维度来将平移操作纳入变换矩阵叫做齐次坐标，反之则被称为仿射变换。

## 变换矩阵的逆

缩放的逆矩阵是对应缩放系数的倒数组成。
旋转矩阵的逆是其倒置，而平移矩阵的逆是相反的操作。
如果能将一个变换矩阵分解为性质清晰的变换矩阵，那么求逆就能大幅度节省效率。

例如使用SVD分解求解逆矩阵。

## 坐标变换

在原点为$\mathbf{p}$、基为$\{\mathbf{u}，\mathbf{v}，\mathbf{w}\}$的坐标系中，坐标$(u, v, w)$:

$$
\mathbf{p} + u\mathbf{u} + v\mathbf{v} + w\mathbf{w}
$$

在二维坐标系中，当尝试将以点e和基u和v组成的坐标系下的坐标$(u_p, v_p)$转换为原点o坐标系下的点$(x_p, y_p)$:

$$
\begin{bmatrix}
x_p \\ y_p \\ 1
\end{bmatrix} = \begin{bmatrix}
1 & 0 & x_e \\ 0 & 1 & y_e \\ 0 & 0 & 1
\end{bmatrix}\begin{bmatrix}
x_u & x_v & 0 \\ y_u & y_v & 0 \\ 0 & 0 & 1
\end{bmatrix}\begin{bmatrix}
u_p \\ v_p \\ 1
\end{bmatrix} = \begin{bmatrix}
x_u & x_v & x_e \\ y_u & y_v & y_e \\ 0 & 0 & 1
\end{bmatrix}\begin{bmatrix}
u_p \\ v_p \\ 1
\end{bmatrix}
$$

也就是：

$$
P_{xy} = \begin{bmatrix}
\mathbf{u} & \mathbf{v} & \mathbf{e} \\ 0 & 0 & 1
\end{bmatrix}P_{uv}
$$