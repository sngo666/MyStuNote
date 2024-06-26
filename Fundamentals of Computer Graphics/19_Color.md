# Chapter-19 Color

我们可以这样认为，光子是光学信息的携带者。每个光子携带固定的能量，由其波长来表示，因而一个光子具有一个特定的波长，其波长$\lambda$和所携带的能量$\triangle E$的关系：

$$
\lambda \triangle E = 1239.9
$$

$\triangle E$以电子伏特为单位测量。

光谱描述了每个波长下有多少能量可用，因此横轴的单位是波长作为观测的方向，相对辐射功率描述里的光子携带的能量总和。
人类对于波长的敏感范围大概是380nm到800nm之间，pbrt中的定义是380~830。

深入探讨光线如何被视网膜捕获并被转换为电信号从而让大脑得到颜色色度上的认知不属于计算机图形学探讨的范畴，但是我们可以从刺激色开始，讨论图形学对于颜色的度量。

Grassman定律总结了基于颜色匹配可能性的一些假设。
对于刺激色A,B,C,D：

* 如果A匹配B,则B匹配A。
* 如果A匹配B且B匹配C，A匹配C
* 如果A匹配B，那么αA匹配αB，α是正比例因子
* 如果A匹配B，C匹配D，且A+C匹配B+D，那么A+D匹配B+C

## 锥形响应

CIE-1931-RGB色彩空间和CIE-1931-XYZ是国际照明委员会(CIE)于1931年创建。
具有正常视力的人嗯眼具有三种感知光的视锥细胞，分别具有短波(420~440)、中波(530~540)和长波(560~580)的光谱灵敏度峰值。因此，与三种视锥细胞的刺激水平相对应的三个参数原则上描述了任何人类颜色感觉。通过三种视锥细胞各自的光谱灵敏度对总光功率谱进行加权，得出三种有效的刺激值。
这三个值构成了光谱的目标颜色的三色规格，表示为“S”、“M”和“L”的三个参数使用名为“LMS 颜色空间”的3维空间来表示，LMS 颜色空间是设计用来量化人类色觉的众多颜色空间之一。

需要注意的是，在某些色彩空间，例如XYZ和LMS中，使用的原色不是真实的颜色，因为其无法在任何光谱中生成。

不同波长的不同混合物组成的两个光源。此类光源可能看起来颜色相同；这种效应被称为“同色异谱”。当此类光源产生相同的三刺激值时，无论光源的光谱功率分布如何，它们对观察者都具有相同的表观颜色。
此外，大多数波长都会刺激两种或全部三种视锥细胞，因为三种视锥细胞的光谱灵敏度曲线重叠。因此，某些三色刺激值在物理上是不可能的，比如一个单单不为零的M值组成的三刺激值。

### 亮度

亮度也许是一个非常感性的描述，在光线充足的情况下判断不同颜色的相对亮度（亮度）时，人类倾向于认为光谱中绿色部分的光比同等功率的红色或蓝色光更亮。因此，描述不同波长的感知亮度的光度函数大致类似于M视锥细胞的光谱灵敏度。

CIE模型利用了这一特性，将Y定义为亮度，而Z被定义为类似于蓝色刺激或S视锥细胞响应，X被定义为非负的视锥细胞相应曲线的混合(这是一种基于线性的组合).因此，XYZ与LMS存在直接的关系，但是又不完全相同。
XYZ的单位的选择通常是任意的，因为可以通过彼此的关系进行计算。

### CIE色彩匹配函数

$\bar{x}(\lambda)$ , $\bar{y}(\lambda)$ 和 $\bar{z}(\lambda)$ 被用于描述产生CIE三刺激值XYZ的三个线性光检测器的光谱灵敏度曲线，
其他观察者例如CIE-RGB空间使用其对应的三组颜色匹配函数，并产生其自身空间的三刺激值。
我们根据光谱辐射亮度$L_{e,\Omega, \lambda}$计算该空间下的三刺激值：

$$
X = \int_\lambda L_{e,\Omega, \lambda}(\lambda)\bar{x}(\lambda)d\lambda \\
$$

同理，也当然就有：

$$
Y = \int_\lambda L_{e,\Omega, \lambda}(\lambda)\bar{y}(\lambda)d\lambda \\
$$

$$
Z = \int_\lambda L_{e,\Omega, \lambda}(\lambda)\bar{z}(\lambda)d\lambda \\
$$

$\lambda$ 代表了等效单色光的波长(以纳米为单位测量)，积分的限值则为人类肉眼可见的波长范围，wikipedia给出的是[380, 780]。
由于Y被定义为亮度，因此$\bar{y}(\lambda)$等同于亮度相应函数$V(\lambda)$。
XYZ颜色空间诞生的缘由有几个：首先是RGB空间的颜色匹配函数并非完全在x轴上方，这对于1931年的积分计算来说有较高的难度，其次Y作为完全提供亮度的一个维度，方便于在创建某些波长的匹配情况中，加减亮度的需求。

### 反射和透射的思考

反射和透射在这一基础上进一步细化，光谱辐射亮度$L_{e,\Omega, \lambda}$被测量物体的光谱反射率(或者透射率)$S(\lambda)$乘以光源的光谱功率分布$I(\lambda)$代替

$$
X = \frac{K}{N}\int_\lambda S(\lambda) I(\lambda) \bar{X}(\lambda)d\lambda \\
$$

$$
Y = \frac{K}{N}\int_\lambda S(\lambda) I(\lambda) \bar{y}(\lambda)d\lambda \\
$$

$$
Z = \frac{K}{N}\int_\lambda S(\lambda) I(\lambda) \bar{z}(\lambda)d\lambda \\
$$

其中N表示为：

$$
N = \int_\lambda I(\lambda)\bar{y}(\lambda)d\lambda
$$

K被定义为比例因子，通常为1或100。并且同样的，$\lambda$被限制在人类肉眼的可见波长范围内。

### CIE-xy 色度图和 CIE-xyY 色彩空间

由于人眼具有三种类型的颜色传感器，可以响应不同的波长范围，因此所有可见颜色的完整图是一个三维图形。然而，颜色的概念可以分为两部分：亮度和色度。例如，白色是明亮的颜色，而灰色被认为是相同白色的不太明亮的版本。换句话说，白色和灰色的色度相同，但亮度不同。
在XYZ色彩空间的基础上，Y作为颜色亮度的度量，同时使用两个派生参数x和y制定颜色的色度：

$$
x = \frac{X}{X + Y + Z}
$$

$$
y = \frac{Y}{X + Y + Z}
$$

$$
z = \frac{Z}{X + Y + Z} = 1- x - y
$$

由此建立xyY色彩空间，对应的X和Z可以计算出：

$$
X = \frac{Y}{y}x
$$

$$
Z = \frac{Y}{y}(1 - x - y)
$$

CIE图上所有可见色度的色域都是以颜色显示的舌形或马蹄形图形。
色域的弯曲边缘称为光谱轨迹，对应于单色光（每个点代表单个波长的纯色调），波长以纳米为单位列出。色域下部的直边称为紫色线。这些颜色虽然位于色域的边缘，但在单色光中没有对应的颜色。图形内部出现饱和度较低的颜色，中心为白色。
如果在色度图上选择任意两点颜色，则两点之间直线上的所有颜色都可以通过这两种颜色混合而形成。两种同样明亮的颜色的等量混合通常不会位于该线段的中点。更一般地说，CIE-xy色度图上的距离并不对应于两种颜色之间的差异程度。

可以看出，给定三个真实源，这些源无法覆盖人类视觉的色域(通过插值公式)。从几何角度来说，色域内没有三个点可以形成包含整个色域的三角形；或者更简单地说，人类视觉的色域不是三角形。
波长平坦的功率谱(每1nm间隔的功率相等)的光对应于点 (x, y) = (1/3, 1/3)。

### 颜色混合

两种或多种颜色相加混合时，所得颜色的x和y色度坐标$(x_{mix}, y_{mix})$可以表示为：

$$
x_{mix} = \frac{\frac{x_1}{y_1}L_1 + \frac{x_2}{y_2}L_2 + ... + \frac{x_n}{y_n}L_n}{\frac{L_1}{y_1} + \frac{L_2}{y_2} + ... + \frac{L_n}{y_n}}, y_{mix} = \frac{L_1 + L_2 + ... + L_n}{\frac{L_1}{y_1} + \frac{L_2}{y_2} + ... + \frac{L_n}{y_n}}
$$

顾名思义，$x$和$y$代表了颜色空间的x和y分量，L代表了对应的亮度。
自然，我们也可以从该公式中观察到两点颜色混合之间的关系：

$$
\frac{L_1}{L_2} = \frac{y_1(x_2 - x_{mix})}{y_2(x_{mix} - x_1)} = \frac{y_1(y_2 - y_{mix})}{y_2(y_{mix} - y_1)}
$$

### RGB空间

CIE-RGB颜色空间是众多RGB颜色空间之一，以一组特定的单色（单波长）原色来区分。

标准使用700nm(红色), 546.1(绿色)和435.8(蓝色)三种单色原色。
这些分别是在另外两种匹配函数为0时该原色所对应的波长(应该不难理解)。
而实际上，人眼实际上可以看到波长高达约810nm的光，但其灵敏度比绿光低数千倍。这些颜色匹配函数定义了所谓的“1931-CIE标准观察者”。
另外需要注意的是，并非指定每个原色的亮度，而是将曲线标准化为在其下方具有恒定的面积。通过指定该区域固定为特定值。
然后，将生成的归一化颜色匹配函数按源亮度$1:4.5907:0.0601$的$r:g:b$比例和源辐射度$72.0962:1.3791:1$的$r:g:b$比例缩放，以再现真实的颜色匹配函数。

$$
\int^{\infin}_0 \bar{r}(\lambda)d\lambda = \int^{\infin}_0 \bar{g}(\lambda)d\lambda = \int^{\infin}_0 \bar{b}(\lambda)d\lambda
$$

同理，根据光谱功率分布$S(\lambda)$得到RGB刺激值:

$$
R = \int^{\infin}_0 S(\lambda)\bar{r}(\lambda)d\lambda
$$

$$
G = \int^{\infin}_0 S(\lambda)\bar{g}(\lambda)d\lambda
$$

$$
B = \int^{\infin}_0 S(\lambda)\bar{b}(\lambda)d\lambda
$$

可以被认为是无限维光谱到三维颜色的投影。
CIE-RGB空间可用于以通常的方式定义色度坐标r和g：

$$
r = \frac{R}{R+G+B}
$$

$$
g = \frac{G}{R+G+B}
$$

非单色原色在所有可见波长上积分，从而产生XYZ坐标，最后产生xy色度图上的点，对于多个点的混合计算也可以使用上文的方式计算。

### RGB转化到XYZ

$$
\begin{bmatrix}
  X \\ Y \\ Z
\end{bmatrix} = \frac{1}{0.17697} \begin{bmatrix}
 0.4900 & 0.3100 & 0.2000 \\ 0.17697 & 0.81240 & 0.01063 \\ 0.0000 & 0.0100 & 0.9900
\end{bmatrix} \cdot \begin{bmatrix}
  R \\ G \\ B
\end{bmatrix}
$$

### CIE-u'v'色度图

由于CIE颜色匹配函数不代表人类锥体的敏感性，色度图上任何两点之间的距离并不能很好地指示这些颜色将被感知的不同程度。
CIE-u′v′色度图提供了感知上更均匀的间距，因此通常优于(x，y)色度图。它是通过应用不同的归一化从(X，Y，Z)三刺激值计算的。

$$
u' = \frac{4X}{X + 15Y + 3Z}
$$

$$
v' = \frac{9Y}{X + 15Y + 3Z}
$$

同理，也可以根据xyY进行转换：

$$
u' = \frac{4x}{-2x + 12y + 3}
$$

$$
v' = \frac{9y}{-2x + 12y + 3}
$$

## 颜色空间

每个颜色都可以用三个数字表示，但是颜色本身仅仅是一个维度，并不代表一个具备三光源的设备，就能够显示光谱中的所有颜色。
出于相同的原因，使用图像编码和计算都不实用。因此，有必要设计其他颜色空间：物理可实现性、高效编码、感知均匀性和直观的颜色规范。

CIE-XYZ颜色空间仍在积极使用，主要用于其他颜色空间之间的转换。它可以看作是一个独立于设备的颜色空间。
可以根据它们与CIE XY Z的关系来定义其他颜色空间，该关系通常由特定变换指定。例如，线性和加性三色显示设备可以通过简单的3×3矩阵转换为CIE-XYZ。当然，也可以指定一些非线性附加转换。

### 建立转换方程

对于显示设备，通过发射单色矢量测量发射光的光谱组成，进而计算x和y的色度坐标。此外白光也应当被测量，其被定义为颜色矢量(1, 1, 1)，通过这四个坐标$(x_{R/G/B/W}, y_{R/G/B/W})$扩展到xyz：

$$
\begin{bmatrix}
  X \\ Y \\ Z
\end{bmatrix} =  \begin{bmatrix}
 x_RS_R & x_GS_G & x_BS_B \\ y_RS_R & y_GS_G & y_BS_B \\ z_RS_R & z_GS_G & z_BS_B
\end{bmatrix} \cdot \begin{bmatrix}
  R \\ G \\ B
\end{bmatrix}
$$

任何给定颜色的亮度都可以通过评估以这种方式构建的矩阵的中间行来计算。

接下来是sRGB的非线性编码：

$$
R_{sRGB} =\begin{cases}
1.055R^{1/2.4} - 0.055 & R > 0.0031308 \\
12.92R & R \le 0.0031308 \\
\end{cases}
$$

$$
G_{sRGB} =\begin{cases}
1.055G^{1/2.4} - 0.055 & G > 0.0031308 \\
12.92G & G \le 0.0031308 \\
\end{cases}
$$

$$
B_{sRGB} =\begin{cases}
1.055B^{1/2.4} - 0.055 & B > 0.0031308 \\
12.92B & B \le 0.0031308 \\
\end{cases}
$$

每个设备都有自己的一组原色和白点，因而这四组值被定位为设备相关。
如果图像是在已知的RGB颜色空间中指定的，则可以首先将其转换为与设备无关的XYZ，然后可以将其转换到将在其上显示的设备的RGB空间。还有其他几个RGB颜色空间定义良好。它们分别由线性矩阵变换和非线性变换组成，类似于前面提到的sRGB颜色空间。非线性变换可以参数化如下：

$$
R_{nonlinear} =\begin{cases}
(1 + f)R^{\gamma} - f & t < R \le 1 \\
sR                    & 0 \le R \le t\\
\end{cases}
$$

$$
G_{nonlinear} =\begin{cases}
(1 + f)G^{\gamma} - f & t < G \le 1 \\
sG                    & 0 \le G \le t\\
\end{cases}
$$

$$
B_{nonlinear} =\begin{cases}
(1 + f)B^{\gamma} - f & t < B \le 1 \\
sB                    & 0 \le B \le t\\
\end{cases}
$$

参数$s,f,t$和$\gamma$指定了一类在各种行业中使用的RGB颜色空间。

### 对立颜色空间(Color Opponent Spaces)

颜色对立空间的特征在于一个通道表示消色差通道(亮度)，以及两个通道编码颜色对立。这些通道通常是红绿色和黄蓝色通道。例如:红色-绿色通道将红色编码为正值，将绿色编码为负值。零值编码一种特殊情况：中性，既不是红色也不是绿色。
这样的设计源于人类先天的特性，人类无法感知同时是红色和绿色，或者黄色和蓝色的颜色。我们没有看到任何类似于红绿或黄蓝色的东西。然而，我们能够感知黄红色（橙色）或绿蓝色等颜色的混合物，因为这些颜色是通过彩色通道编码的。

计算机图形学最相关的颜色对抗系统是CIE 1976$L^*a^*b^*$颜色模型。它是一个在感知上或多或少均匀的颜色空间，除其他外，对色差的计算很有用。它也被称为CIELAB。CIELAB的输入是刺激（X，Y，Z）三刺激值以及由已知光源$(X_n，Y_n，Z_n)$照明的漫射白色反射表面的三刺激值。因此，CIELAB超越了普通的颜色空间，因为它在已知照明的背景下考虑了一块颜色。

$$
\begin{bmatrix}
  L^* \\ a^* \\ b^*
\end{bmatrix} =  \begin{bmatrix}
 0 & 116 & 0 & −16 \\ 500 & -500 & 0 & 0 \\ 0 & 200 & -200 & 0
\end{bmatrix} \cdot \begin{bmatrix}
  f(X/X_n) \\ f(Y/Y_n) \\ f(Z/Z_n) \\ 1
\end{bmatrix}
$$

$$
f(r) = \begin{cases}
  \sqrt[3]{r}             & r > 0.008856\\
  7.787r + \frac{16}{116} & r \le 0.008856
\end{cases}
$$

从这个公式可以看出，彩色通道确实取决于亮度Y。尽管这在感知上是准确的，但这意味着我们无法在色度图中绘制$a^*$和$b^*$的值。
色差公式可以直接通过计算欧氏距离来获得：

$$
\triangle E^*_{ab} = [(\triangle L^*)^2 + (\triangle a^*)^2 + (\triangle b^*)^2]^{\frac{1}{2}}
$$

最后，CIE-LAB和XYZ之间的逆变换由下式给出:

$$
X = X_n\begin{cases}
  (\frac{L^*}{116} + \frac{a*}{500} + \frac{16}{116})^3 & if L^* > 7.9996\\
  \frac{1}{7.787}(\frac{L^*}{116} + \frac{a*}{500}) & if L^* \le 7.9996
\end{cases}
$$

$$
Y = Y_n\begin{cases}
  (\frac{L^*}{116} + \frac{16}{116})^3 & if L^* > 7.9996\\
  \frac{1}{7.787}\frac{L^*}{116} & if L^* \le 7.9996
\end{cases}
$$

$$
Z = Z_n\begin{cases}
  (\frac{L^*}{116} - \frac{b^*}{200} + \frac{16}{116})^3 & if L^* > 7.9996\\
  \frac{1}{7.787}(\frac{L^*}{116} - \frac{b^*}{200}) & if L^* \le 7.9996
\end{cases}
$$

## 色度自适应

我们在日常生活中遇到的观看环境范围很大，从阳光到星光，从烛光到荧光。照明条件不仅在存在的光量上构成非常大的范围，而且在发射光的颜色上变化很大。

人类的视觉系统通过一个称为适应的过程来适应环境中的这些变化。可以区分三种不同类型的适应，即光适应、暗适应和色适应。

* 光适应是指当我们从非常黑暗的环境转移到非常明亮的环境时发生的变化。当这种情况发生时，起初我们会被光线弄得眼花缭乱，但很快我们就会适应新的情况，并开始区分环境中的物体。
* 黑暗适应指的是相反的情况——当我们从光明的环境进入黑暗的环境时。起初，我们看到的很少，但经过一段时间后，细节就会开始显现。适应黑暗所需的时间通常比适应光线所需的要长得多。
* 色彩适应是指我们适应并在很大程度上忽略照明颜色变化的能力。

从本质上讲，色彩适应是大多数现代相机上可用的白平衡操作的生物学等效物。人类视觉系统有效地规范了观看条件，以呈现相当一致的视觉体验。

尽管我们能够在很大程度上忽略观看环境的变化，但我们不能完全忽略。
例如，在阳光明媚的日子里，颜色看起来比在阴天更丰富多彩。尽管外观发生了变化，但我们并不认为物体反射率本身实际上改变了其物理特性。因此，我们了解到照明条件已经影响了整体颜色外观。尽管如此，颜色恒定性确实适用于彩色内容。

色自适应的计算模型往往侧重于锥体中的增益控制机制。最简单的模型之一假设每个圆锥体独立地适应其吸收的能量。这意味着不同的锥体类型根据被吸收的光的光谱而不同地适应。
von-Kries适应作为最简单的色自适应模型，提出由观看环境决定的独立增益控制锥信号：

$$
L_a = \alpha L,\\
M_a = \beta M,\\
S_a = \gamma S,
$$

自适应照明可以在场景中的白色表面上测量。在理想情况下，这将是一个朗伯曲面。在数字图像中，自适应照明也可以近似为场景的最大三刺激值。以这种方式测量或计算的光是由$(L_w，M_w，S_w)$给出的自适应白色。Von-Kries自适应简单地通过倒数进行缩放:

$$
\begin{bmatrix}
  L_a \\ M_a \\ S_a
\end{bmatrix} =  \begin{bmatrix}
 \frac{1}{L_w} & 0 & 0  \\ 0 & \frac{1}{M_w} & 0  \\ 0 & 0 & \frac{1}{S_w}
\end{bmatrix} \cdot \begin{bmatrix}
  L \\ M \\ S
\end{bmatrix}
$$

通过级联两个颜色自适应计算来实现。本质上，前面提到的von Kries变换划分出了适应的光源——在我们的例子中，就是日光照明。如果我们随后将白炽光源相乘，我们就计算出了相应的颜色。如果两个发光体由$(L_{w,1}，M_{w,1},S_{w,1})$和$(L{_w,2}，M_{w,2},S_{w,2})$给出，则对应的颜色$(L_{c}，M_{c},S_{c})$由下式给出:

$$
\begin{bmatrix}
  L_c \\ M_c \\ S_c
\end{bmatrix} =  \begin{bmatrix}
 L_{w,2} & 0 & 0  \\ 0 & M_{w,2} & 0  \\ 0 & 0 & S_{w,2}
\end{bmatrix}\begin{bmatrix}
 \frac{1}{L_{w,1}} & 0 & 0  \\ 0 & \frac{1}{M_{w,1}} & 0  \\ 0 & 0 & \frac{1}{S_{w,1}}
\end{bmatrix} \cdot \begin{bmatrix}
  L \\ M \\ S
\end{bmatrix}
$$

色彩自适应在渲染中的重要性在于，我们离考虑观察者的观看环境又近了一步，而不必通过调整场景和重新渲染图像来进行校正。相反，我们可以对场景进行建模和渲染，然后作为图像后处理，对观看环境的照明进行校正。然而，为了确保白平衡不会引入伪影，重要的是要确保图像渲染为浮点格式。如果呈现为传统的8位图像格式，则色度自适应变换可能放大量化误差。
《Color imaging: fundamentals and applications》提到了更多的色度自适应变换。

## 颜色外观

虽然色度学使我们能够以独立于设备的方式准确地指定和传达颜色，而色度自适应使我们能够预测照明变化中的颜色匹配，但这些工具仍然不足以描述颜色的实际外观。
为了预测物体的实际感知，我们需要了解更多关于环境的信息，并将这些信息考虑在内。人类的视觉系统不断地适应环境，这意味着对颜色的感知将受到这种变化的强烈影响。颜色外观模型考虑了刺激本身的测量以及观看环境。这意味着所得到的颜色描述与观看条件无关。
《Color appearance models》给出了更加专业的指导。
