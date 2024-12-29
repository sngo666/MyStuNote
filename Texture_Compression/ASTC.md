# ASTC (Adaptive Scalable Texture Compression)

ASTC纹理可以是2D或3D。
ASTC纹理可以使用高动态范围或低动态范围进行编码。可以选择使用RGB通道的sRGB传递函数来指定LDR图像。

每个块使用128KB的固定内存，用以表示不同大小的块。

该格式的全局配置数据如下：

* 块尺寸（2D 或 3D）
* 块占用空间大小
* sRGB 输出是否启用

每个数据块指定的数据如下：

* 纹素权重网格大小
* 纹素重量范围
* 纹素权重值
* 分区数
* 分区模式索引
* 颜色端点模式（包括 LDR 或 HDR 选择）
* 为端点数据着色
* 平面数
* 平面到通道分配

![Alt](./res/astc_layout.png#pic_center)

由于“texel weight data”字段的大小是可变的，因此“more config data”字段和“color endpoint data”字段显示的位置仅具有代表性，而不是固定的。"Block mode"指出了这一字段的具体编码方式。


