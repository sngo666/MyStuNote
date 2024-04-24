# chapter-07 Viewing

## Viewport 变换

将正方形$[-1, 1]$映射到矩形$[-0.5, n_x-0.5] * [-0.5, n_y-0.5]$上：
这里的n指的是屏幕上像素的个数，这里的意思是将像素看作是长度为1个单位的方框的中心位置，并且坐标系从左下方的像素，也就是其所在的正方格的中心开始。

$$
\begin{bmatrix}
x_{screen} \\ y_{screen} \\ 1
\end{bmatrix} = \begin{bmatrix}
\frac{n_x}{2} & 0 & \frac{n_x-1}{2} \\
0 & \frac{n_y}{2} & \frac{n_y-1}{2} \\
0 & 0 & 1
\end{bmatrix} \begin{bmatrix}
x_{canonical} \\ y_{canonical} \\ 1
\end{bmatrix}
$$

$$
M_{vp} = \begin{bmatrix}
\frac{n_x}{2} & 0 & 0 & \frac{n_x-1}{2} \\
0 & \frac{n_y}{2} & 0 & \frac{n_y-1}{2} \\
0 & 0 & 1 & 0 \\
0 & 0 & 0 & 1 
\end{bmatrix}
$$

## 正交投影变换

$$
(l, b, n) --> (left, bottom, near) \\
(r, t, f) --> (right, top, far)  \\
n>f
$$


将一个空间内投影到固定的标准空间内：

$$
M_{oth} = \begin{bmatrix}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} &  0 & -\frac{t+b}{t-b} \\
0 &  0 & \frac{2}{n-f} & -\frac{n+f}{n-f} \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

最后，先执行正交投影变换，再执行viewport变换，就能够将一个空间转换成屏幕上的像素：

$$
\begin{bmatrix}
x_{pixel} \\ y_{pixel} \\ z_{canonical} \\ 1
\end{bmatrix} = (M_{vp}M_{orth}) \begin{bmatrix}
  x \\ y \\ z \\ 1
\end{bmatrix}
$$

在执行完这一系列的计算，z坐标集中在-1到1之间，通过进一步的z-buffer来获取屏幕上的像素。

## 摄像机变换

由上一章的坐标变换，很容易完成从原点坐标系到摄像机坐标系的转换，但是需要注意的是，摄像机的朝向也就是观察方向指向-z轴，面向-z方向，向上为y轴，向右为x轴。

$$
M_{cam} = \begin{bmatrix}
x_u & y_u & z_u & 0 \\
x_v & y_v & z_v & 0 \\
x_w & y_w & z_w & 0 \\
0 & 0 & 0 & 1
\end{bmatrix} \begin{bmatrix}
  1 & 0 & 0 & -x_e \\
  0 & 1 & 0 & -y_e \\
  0 & 0 & 0 & -z_e \\
  0 & 0 & 0 & 1
\end{bmatrix}
$$

最后，完成这个变换：

$$
M = M_{vp}M_{orth}M_{cam}
$$

## 射影(projective)变换

如果我们允许变换矩阵底行中的任何值，则它允许实现更广泛的转换，导致w取除了1以外的值。
感觉说了，又什么也没说，就是说一下可以利用w作为第四个维度来充当一个公共分母。

## 透视投影

$$
P = \begin{bmatrix}
  n & 0 & 0 & 0 \\
  0 & n & 0 & 0 \\
  0 & 0 & n+f & -fn \\
  0 & 0 & 1 & 0
\end{bmatrix}
$$

也就是：

$$
P\begin{bmatrix}
  x \\ y \\ z \\ 1
\end{bmatrix} = \begin{bmatrix}
  \frac{nx}{z} \\ \frac{ny}{z} \\ n+f - \frac{nf}{z} \\ 1
\end{bmatrix}
$$

当原先在视锥中z轴上位于$\frac{n+f}{2}$的点，可以发现，离摄像机是更远的。

P逆矩阵为：

$$
P^{-1} = \begin{bmatrix}
  f & 0 & 0 & 0 \\
  0 & f & 0 & 0 \\
  0 & 0 & 0 & fn \\
  0 & 0 & -1 & n+f
\end{bmatrix}
$$

将一个视锥的空间投影到一个正投影空间内，再进一步地进行等比例的缩放和平移就可以进行z-buffer处理了，因此我们的总步骤：

$$
M = M_{vp}M_{orth}PM_{cam}
$$

作为一个对物体的总的投影变换，一些API例如opengl会让用户指定n和f为相应的绝对值。

## 视野空间

通过一组数来定义摄像机空间前的一块屏幕：$(l, r, b, t, n)$

这里摄像机从-z轴出发，恰好经过屏幕的正中心点，也就有了

$$
\tan\frac{θ}{2} = \frac{t}{|n|}
$$

θ是屏幕上边界中点到观察点和下边界中点到观察点组成的夹角。
