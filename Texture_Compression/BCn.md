# BCn家族纹理压缩算法

## BC1

BC1~BC3也被成为S3TC纹理压缩格式。
BC1相当于DirectX9中称为DXT1的压缩纹理格式。对于每一个4X4的像素块，BC1使用64bit的数据用以描述压缩。在这基础上存在细微的分支，即分别支持线性和sRGB颜色空间

$$
\begin{align*}
c0_{\mathit{lo}}, c0_{\mathit{hi}}, c1_{\mathit{lo}}, c1_{\mathit{hi}}, \mathit{bits}_0, \mathit{bits}_1, \mathit{bits}_2, \mathit{bits}_3
\end{align*}
$$

$$
\begin{align*}
\mathit{c}_0 & = c0_{\mathit{lo}} + c0_{\mathit{hi}} \times 256 \\
\mathit{c}_1 & = c1_{\mathit{lo}} + c1_{\mathit{hi}} \times 256 \\
\mathit{bits} & = \mathit{bits}_0 + 256 \times (\mathit{bits}_1 + 256 \times (\mathit{bits}_2 + 256 \times \mathit{bits}_3))
\end{align*}
$$

显然，$\mathit{color}_0$和$\mathit{color}_1$都是16位无符号RGB数字，这是非常规的，因为它们被定义为RGB565。

剩下的32bits用来表示块中的每个像素，每个像素因而被分配2bits，使用其为块中（x，y）位置上的字节提取一个2位的控制代码，

$$
\begin{align*} \mathit{code}(x,y) & = \mathit{bits}[2\times (4\times y+x)+1\ \dots\ 2\times(4\times y+x)+0] \end{align*}
$$

|                    Texel value                     |                 Condition                 |
| :------------------------------------------------: | :---------------------------------------: |
|                      $RGB_0$                       | $color_0$ > $color_1$ and code(x, y) = 0  |
|                      $RGB_1$                       | $color_0$ > $color_1$ and code(x, y) = 1  |
| $(2\times \mathit{RGB}_0 + \mathit{RGB}_1)\over 3$ | $color_0$ > $color_1$ and code(x, y) = 2  |
|     $(\mathit{RGB}_0 + 2\times RGB_1)\over 3$      | $color_0$ > $color_1$ and code(x, y) = 3  |
|                      $RGB_0$                       | $color_0$ <= $color_1$ and code(x, y) = 0 |
|                      $RGB_1$                       | $color_0$ <= $color_1$ and code(x, y) = 1 |
|      $(\mathit{RGB}_0+\mathit{RGB}_1)\over 2$      | $color_0$ <= $color_1$ and code(x, y) = 2 |
|                       BLACK                        | $color_0$ <= $color_1$ and code(x, y) = 3 |

```cpp
color_0 = 00
color_1 = 01
color_2 = 10
color_3 = 11
```

算术运算是按分量进行的，黑色指的是红、绿、蓝均为0的颜色。
可以看出，在这种编码模式下，提供了两种编码模式，控制模式的开关通过两种颜色相对大小控制，这是一种非常原始但聪明的做法。第一种编码模式提供了四种颜色的等差插值，第二种为了纳入黑色，牺牲了颗粒度。如果算上有限的对于透明度通道的编码，那就是总共三种。
可以看出，BC1在有限的bit数内提供了非常粗糙的编码，首先除了黑白两种颜色无法提供任何具体的灰度值，透明度也只有0和1之分。

| Alpha value |                 Condition                 |
| :---------: | :---------------------------------------: |
|     0.0     | $color_0$ <= $color_1$ and code(x, y) = 3 |
|     1.0     |                 otherwise                 |

Direct3D9和10中都存在此格式：
· 在Direct3D9中，BC1 格式被称为 D3DFMT_DXT1。
· 在Direct3D10中，BC1 格式由DXGI_FORMAT_BC1_UNORM或DXGI_FORMAT_BC1_UNORM_SRGB表示。

## BC2

每个4×4纹理块由 64 位未压缩的 Alpha 图像数据和 64 位 RGB 图像数据组成。

每个RGB图像数据块都按照BC1格式编码，但两个代码位始终使用非透明编码。换句话说，无论$color_0$和$color_1$的实际值是多少，它们都被视为$color_0$>$color_1$。
BC2相较于BC1的提升只体现于使用额外的64位内存来存储alpha通道，相当奢侈。
BC2从最初的S3TC开始就存在了，但我从未在实际中使用过。我猜它之所以还存在的主要原因是，如果你已经支持BC1和BC3，那么它的硬件成本就会很低，所以尽管实际使用很少，但它的支持成本足够低，以至于没有人游说将其淘汰。

## BC3

BC3和BC2类似，仅仅针对alpha通道进行了重新编码，每个allpha图像数据块编码为8个字节的序列。
使用两个端点值$alpha_0$和$alpha_1$，每个占用8bit，剩下48bit表示每16个像素。

|                         Alpha value                          |                 Condition                 |
| :----------------------------------------------------------: | :---------------------------------------: |
|                          $alpha_0$                           |              code(x, y) = 0               |
|                          $alpha_1$                           |              code(x, y) = 1               |
| $(6\times\mathit{alpha}_0 + 1\times\mathit{alpha}_1)\over 7$ | $alpha_0$ > $alpha_1$ and code(x, y) = 2  |
| $(5\times\mathit{alpha}_0 + 2\times\mathit{alpha}_1)\over 7$ | $alpha_0$ > $alpha_1$ and code(x, y) = 3  |
| $(4\times\mathit{alpha}_0 + 3\times\mathit{alpha}_1)\over 7$ | $alpha_0$ > $alpha_1$ and code(x, y) = 4  |
| $(3\times\mathit{alpha}_0 + 4\times\mathit{alpha}_1)\over 7$ | $alpha_0$ > $alpha_1$ and code(x, y) = 5  |
| $(2\times\mathit{alpha}_0 + 5\times\mathit{alpha}_1)\over 7$ | $alpha_0$ > $alpha_1$ and code(x, y) = 6  |
| $(1\times\mathit{alpha}_0 + 6\times\mathit{alpha}_1)\over 7$ | $alpha_0$ > $alpha_1$ and code(x, y) = 7  |
| $(4\times\mathit{alpha}_0 + 1\times\mathit{alpha}_1)\over 5$ | $alpha_0$ <= $alpha_1$ and code(x, y) = 2 |
| $(3\times\mathit{alpha}_0 + 2\times\mathit{alpha}_1)\over 5$ | $alpha_0$ <= $alpha_1$ and code(x, y) = 3 |
| $(2\times\mathit{alpha}_0 + 3\times\mathit{alpha}_1)\over 5$ | $alpha_0$ <= $alpha_1$ and code(x, y) = 4 |
| $(1\times\mathit{alpha}_0 + 4\times\mathit{alpha}_1)\over 5$ | $alpha_0$ <= $alpha_1$ and code(x, y) = 5 |
|                             0.0                              | $alpha_0$ <= $alpha_1$ and code(x, y) = 6 |
|                             1.0                              | $alpha_0$ <= $alpha_1$ and code(x, y) = 7 |

## BC4

BC4~BC5被称为RGTC纹理压缩格式。
在BC4中每个单通道数据可被编码为64位无符号模式和有符号模式，类似于BC3的alpha通道，使用两个8bit界定两个基本色$RED_0$和$RED_1$，范围为[0，255]。

|                         R value                          |               Condition               |
| :------------------------------------------------------: | :-----------------------------------: |
|                         $RED_0$                          |            code(x, y) = 0             |
|                         $RED_1$                          |            code(x, y) = 1             |
| $(6\times\mathit{RED}_0 + 1\times\mathit{RED}_1)\over 7$ | $RED_0$ > $RED_1$ and code(x, y) = 2  |
| $(5\times\mathit{RED}_0 + 2\times\mathit{RED}_1)\over 7$ | $RED_0$ > $RED_1$ and code(x, y) = 3  |
| $(4\times\mathit{RED}_0 + 3\times\mathit{RED}_1)\over 7$ | $RED_0$ > $RED_1$ and code(x, y) = 4  |
| $(3\times\mathit{RED}_0 + 4\times\mathit{RED}_1)\over 7$ | $RED_0$ > $RED_1$ and code(x, y) = 5  |
| $(2\times\mathit{RED}_0 + 5\times\mathit{RED}_1)\over 7$ | $RED_0$ > $RED_1$ and code(x, y) = 6  |
| $(1\times\mathit{RED}_0 + 6\times\mathit{RED}_1)\over 7$ | $RED_0$ > $RED_1$ and code(x, y) = 7  |
|                         $RED_0$                          |            code(x, y) = 0             |
|                         $RED_1$                          |            code(x, y) = 1             |
| $(4\times\mathit{RED}_0 + 1\times\mathit{RED}_1)\over 5$ | $RED_0$ <= $RED_1$ and code(x, y) = 2 |
| $(3\times\mathit{RED}_0 + 2\times\mathit{RED}_1)\over 5$ | $RED_0$ <= $RED_1$ and code(x, y) = 3 |
| $(2\times\mathit{RED}_0 + 3\times\mathit{RED}_1)\over 5$ | $RED_0$ <= $RED_1$ and code(x, y) = 4 |
| $(1\times\mathit{RED}_0 + 4\times\mathit{RED}_1)\over 5$ | $RED_0$ <= $RED_1$ and code(x, y) = 5 |
|                       $RED_{min}$                        | $RED_0$ <= $RED_1$ and code(x, y) = 6 |
|                       $RED_{max}$                        | $RED_0$ <= $RED_1$ and code(x, y) = 7 |

在有符号的场景，区别在于：

$$
RED_{min} = -1.0 \\
RED_{max} = 1.0
$$

对于其他的数，除以127.0进行一一映射。

对于带符号的red0和red1值：当red0 = -127和red1 = -128时，上面的表达式red0 > red1和red0 ≤ red1被视为未定义（读取：可能因实现而异）。这是因为，如果在比较之前将red0重新映射到-127以减少硬件解压缩器的延迟，则表达式将反转其逻辑。签名红绿格式的编码器应避免对red0 = -127和red1 = -128的块进行编码。

## BC5

每个4×4纹素块由64位压缩的无符号红色图像数据组成，后跟64位压缩的无符号绿色图像数据。
压缩红色的前64位的解码方式与上面的BC4 unsigned完全相同。压缩绿色的第二个64位的解码方式与上面的无符号BC4完全相同，只是第二个块的解码值R被视为最终的绿色值G。

由于解码后的纹素具有红绿格式，因此纹素的最终RGBA值为(R, G, 0, 1)。
对有符号的版本，采取的编码方式与BC4一样。

