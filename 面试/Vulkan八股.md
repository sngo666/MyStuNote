# Vulkan八股

## 设计思想

### Vulkan设计思想

**强调开发者自己控制一切**: 开发者可以自由的调度渲染流程和管理同步以及内存分配和释放。这些开发者都需要手动完成并且优化因为开发者本身就有这更上层的信息来判断怎么完成优化，而不是像传统图形API一样在驱动程序当中通过开发者调用API模式来揣测并且推断相关操作的优化方法。
**多线程友好**: 利用现在的普遍的多核CPU来避免传统图形API的单线程模型造成的CPU瓶颈问题。抛弃传统图形API中依赖绑定到基于单个线程上的Context概念。而现代图形API(Vulkan,Metal,Dx12)全部是通过Command Buffer来录制Command并通过Queue向GPU提交命令，可以多个线程同时提交录制好Command的Command Buffer，同时将同步丢给开发者自己处理，开发者拥有更高层次的信息以判断更好的同步时机。
**强调复用**: 在Vulkan中大多数组件都可以复用比如资源(Buffer或者Image)或者Command以及描述符等等。
**驱动轻量化**

此外，传统图形API使用单一全局状态，而Vulkan则是使用基于对象的状态

### vk新特性

#### bindless

bindless纹理可以把大量的纹理以数组的形式送入着色器中，且可以在绑定之后修改描述符，且动态数量，且不像Texture2DArray那样对同一组内的纹理格式有严格的要求。

**实现流程**：

1. 创建Bind Less描述符池，与常规的创建Descriptor Pool没有什么区别，只是将`flag`设定为`VK_DESCRIPTOR_POOL_CREATE_FREE_UPDATE_AFTER_BIND_BIT`，但这不是必须的。
2. 创建layout时，要保证flag与pool的创建时flag对齐，否则会报错。与常规创建layoutCI不同的是，需要创建一个`VkDescriptorSetLayoutBindingFlagsCreateInfoEXT`，并让layoutCI的pNext指向其。

```cpp
bindLessLayoutCI.flags = VK_DESCRIPTOR_SET_LAYOUT_CREATE_UPDATE_AFTER_BIND_POOL_BIT;
VkDescriptorBindingFlags  bindless_flags = VK_DESCRIPTOR_BINDING_PARTIALLY_BOUND_BIT|VK_DESCRIPTOR_BINDING_UPDATE_AFTER_BIND_BIT;//VK_DESCRIPTOR_BINDING_VARIABLE_DESCRIPTOR_COUNT_BIT_EXT;
std::vector<VkDescriptorBindingFlags> binding_flags.resize(bindings.size());
for(auto& flag:binding_flags){
     flag = bindless_flags;
}
VkDescriptorSetLayoutBindingFlagsCreateInfo extended_info{VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_BINDING_FLAGS_CREATE_INFO,
                                                                  nullptr};
extended_info.bindingCount = bindings.size();
extended_info.pBindingFlags = binding_flags.data();
bindLessLayoutCI.pNext = &extended_info;
vkCreateDescriptorSetLayout(******);
```

接下来是按正常流程创建set了，描述符集创建好了以后，应该有一个vkUpdateDescriptorSet,来吧imageInfo和某一个描述符建立起联系。

渲染器中有一个资源池，资源池中某个纹理的句柄(其实就是索引)作为这个纹理在bindless描述符集中的描述符索引。
在加载模型时，到了创建纹理的步骤，我都会将这个资源的句柄存到一个bindless队列里面，在每次渲染结束以后，在vkSubmitInfo填写之前，我在这里遍历这个队列，把描述符更新上去

### vulkan实现多线程

k拥有多个physical device，例如gpu或者其他图形处理器，只要是同一个physical device group中的physical device，就可以联合起来一起来创建出一个device。，而每个physical device上又有多个queue，这些queue属于三种queuefamily(grahpic，compute，translate)。
每个queue family都会创建相应的command pool，而command pool分别创建cmd buffer。这些buffer在绑定指令之后，被提交到任意一个queue上，这样就完成了cpu同gpu搭建的桥梁。

### vulkan并行粒度

vulkan提出的规范，**一个renderpass必须被包含在同一个cmdbuffer中**。

这意味着我们不能将一个renderpass中的drawcall拆分到多个cmdbuffer中，而移动端游戏为了带宽的最小化，都是尽量减少合并renderpass的，vulkan考虑了这个问题，并引入secondary commandbuffer的概念来解决这个问题。

Secondary command buffer是中特殊的command buffer：

1. 它内部不能执行renderpass相关的操作，只能执行drawcall相关的API。
2. 它不能直接submmit给queue，只能被正常的command buffer（或称为primary command buffer）通过vkexecutecmdbuffer的形式执行
3. Vkexecutecmdbuffer必须在该primary commandbuffer结束记录状态前（vkendcommandbuffer）执行

我们的设计支持两种类型的并行：

1. **整个pass级别的并行**：每个cmdbuffer里面封装1个或几个renderpass，renderpass完整的嵌入在一个cmdbuffer里面。因为renderpass之间相互隔离，它的实现最为简单，每个thread上就是正常的启动，结束一个pass和drawcall。但是当一个renderpass里面的drawcall太多时，我们就必须实现drawcall级别的并行。
2. **支持draw call级别的并行**：vulkan API限制renderpass不能跨越cmdbuffer，需要依靠`secondary command buffer`支持。

## 实现

### gamma矫正

人眼对亮度的反应是非线性增长的，人眼可感知的亮度范围为$10^{-2} — 10^6 cd/m^2$(坎德拉每平方米)，且人眼对亮度变化的反应随着光的增加而减弱。若用I表示亮度，S表示人眼反应：$S = a\cdot \log I$，其中a为常数。
相机的编码， 八位也就是255个编码只能表示2.55倍的亮度范围，如此小的亮度范围显然不符合观众对亮度的要求，这 255 个编码对于亮度的变化来说预算完全不足。
Gamma校正就出现了，Gamma本质上为幂函数:$C = I^{1/\gamma}$，其中gamma为常数。通过Gamma校正后的非线性编码更能体现人眼的识别范围，并且节省编码空间。
人眼对暗部变化的感知更为明显。人眼对反射率为18%的灰色的感觉是50%亮度。

CRT是把电能转换成光能的一种设备，CRT的特性决定了从电压到亮度的转换是非线性。CRT的Gamma特性正好与人眼识别光线的特性相反。这样 CRT 显示器上面的画面可以完美再现所拍摄物体的亮度和颜色。这也就是为什么CRT会看上去更加鲜艳的原因。

### HDR

HDR(High Dynamic Range, 高动态范围)指的是：显示器被限制为只能显示值为0.0到1.0间的颜色，但是在光照方程中却没有这个限制。通过使片段的颜色超过1.0，我们有了一个更大的颜色范围。有了HDR，亮的东西可以变得非常亮，暗的东西可以变得非常暗，而且充满细节。
一般，当存储在帧缓冲(Framebuffer)中时，亮度和颜色的值是默认被限制在0.0到1.0之间的。由于大量片段的颜色值都非常接近1.0，在很大一个区域内每一个亮的片段都有相同的白色。解决这个问题的一个方案是减小光源的强度从而保证场景内没有一个片段亮于1.0。然而这并不是一个好的方案.

HDR渲染允许我们用更大范围的颜色值渲染(具体地，对于材质的读取使用更宽的，比如32位)从而获取大范围的黑暗与明亮的场景细节，最后将所有HDR值转换成在[0.0, 1.0]范围的LDR(Low Dynamic Range,低动态范围)。转换HDR值到LDR值得过程叫做色调映射(Tone Mapping)

**Reinhard色调映射**:平均地将所有亮度值分散到LDR上

```glsl
vec3 mapped = hdrColor / (hdrColor + vec3(1.0));
```

**Exposure曝光映射**:在白天使用低曝光，在夜间使用高曝光，就像人眼调节方式一样。有了这个曝光参数，我们可以去设置可以同时在白天和夜晚不同光照条件工作的光照参数

```glsl
vec3 mapped = vec3(1.0) - exp(-hdrColor * exposure);
```

### Shadow Mapping

shadow mapping性能主要依赖于定义的尺寸，一般来说是2048个像素，在移动端上出于性能的考量可能会定义一个较小的尺寸，比如1024.
**第一次 pass**：生成阴影贴图。
以该尺寸定义一个Image和对应的imageview，这些都是单层且线性优化的方式存储，同时不附带多层采样和color_attachment，仅设置单条depth_attachment并定义store operation.这就是生成shadow map。
**第二次 pass**：正式渲染场景。
用第一次pass里面的矩阵M将三维点变换为二维坐标和深度， 将深度与第一次pass存下来的阴影贴图对应点的深度进行对比，若大于，则认为其处于阴影中。

存在一些问题：

1. 点光源应该是各个方向都有光，而上图中表现得更像是聚光灯，如果要实现点光源的效果，一个常用的方法是：分别朝六个方向生成阴影贴图，然后构成一个立方体贴图 (Cube Map)
2. 由于深度的数值精度和阴影贴图分辨率都有限，所以在进行深度比较的时候，有可能会出现Z-fighting的现象，所以需要在比较时添加偏差，称之为**Depth Bias**
3. 由于阴影贴图分辨率是有限的，每个像素占据一定大小，并且离光源越远，每个像素覆盖的片元就越多，并且涉及到采样和重采样，那么就可能导致阴影产生锯齿**Aliasing**

解决方案：

1. 对于**Depth Bias**，设定一个基于坡度的bias，过小会导致z-fighting, 过大会导致影子飞悬。进一步地，有成果表明不使用最靠近光源的片元深度作为阴影贴图，而使用第二近的片元深度 (Second-Depth)或者 中间深度 (Midpoint) 
2. 对于**Aliasing**，即使可以把阴影贴图的分辨率和深度设置得很大，同时增加内存负载，但仍然存在采样和重采样的问题。

针对这样的问题，简单介绍三种方式，进行处理：
**Fitting**:针对光照范围和视角范围的不完全重叠，例如通过片元级别、几何级别、包围体级别三个等级的遮挡关系，进行选择性渲染以实现的优化。这是通过选择显著减少渲染区域以将运算负载集中在采样和重采样上完成。

**Warp**：近处的精度需求高于远处，但是在生成阴影贴图时没有考虑这一点，对变换矩阵M做了一定的扭曲（warp），使近处的精度更高，所以看上去没有严重的锯齿。Marc等人于2002年首先提出了透视阴影贴图（PSM）的方法：
先将场景（包括光源）用相机矩阵（MVP矩阵）变换到规范视域体，再xyz同除w变换到post-perspective space（指归一化设备空间NDC），此时再从光源的位置，执行一次正交投影变换，生成阴影贴图（此方法与常规方法的**区别**在于：用相机 VP 矩阵直接替代了光源的VP矩阵）
转到post-perspective space 后，光源为定向平行光，且相机的view vector与light vector垂直，此时可消去所有因为透视带来的锯齿。

**Partition**：一种最流行的Partition方法 z-partition：

1. 沿着z轴将相机的视锥体划分为多个子视锥体，然后对每个子视锥体单独计算阴影贴图。
最简单的切分方法就是沿z轴均匀切分，但是越靠近near平面，切分区域应该更小才合理。所以另一种是按对数切分。
此切分方法会在 near 面附近分配很多的“分辨率”，但是实际中，相机的near平面附近往往没有物体，所以可将以上两种方法合并起来，这也是 CSM 采取的切分方式，通过一个lerp用于控制权重。

2. 到切分区域的z值之后，就可以结合视场角计算出每个区域的角点，然后需要计算出PSC（潜在阴影遮挡者）的最小包围盒，包围盒的边平行于光线的投影视锥体（由于是正交投影，所以是一个长方体），可以得到包围盒的最大值和最小值.
3. 在最终进行场景渲染时，需要先将当前片元的深度与前面的 N 个区域的 z 值范围进行比较，以确认落在哪个区域。

**重采样问题**：阴影的边界呈现突变，称之为“硬阴影”，并且如果阴影贴图的分辨率不够，很可能出现锯齿。实际生活中的光源都有尺寸，所以阴影的边缘会有一个强度渐变的现象，称之为软阴影。真实的阴影包含本影和半影。
书接下文。

### PCF

解决阴影贴图的重采样问题主要是通过**滤波**（Filtering）解决三件事：
**减小重采样的错误**、**平滑阴影边缘**、**模拟软阴影**

对于阴影贴图的滤波，则是先把片元的深度和阴影贴图对应点进行比较，得到离散的 0-1 值，再对这些 0-1 值进行滤波，用局部多个点来确定某一点的“阴影程度”，此时得到的值可以是 0-1 之间的连续值，这种方法称为PCF**百分比近似滤波**

传统的**盒滤波**会导致条纹现象。
**泊松圆盘滤波**：此滤波从一个圆盘范围内取少量服从泊松分布点作为采样点，在取这些点的时候可以用最近邻采样或者双线性采样，相比方形的滤波核，此采样点不那么规则。
在实际使用时，如果使用泊松圆盘滤波，则需要预计算各个点的位置存储在lookup纹理或者 uniform 数组里面.

### PCSS

PCF的滤波半径决定了阴影边缘的柔和程度，一般来说滤波半径越大则柔和，相反阴影边缘越锐利.
由此启发了PCSS（Percentage Close Soft Shadow）软阴影算法的核心原理，对于那些我们需要实现软阴影的边缘用更大的滤波半径进行滤波，而对于那些不需要太过柔和阴影边缘的地方，则采用更小的滤波半径，这是一种自适应滤波半径的机制。
PCSS的本质，概括起来就是自适应PCF滤波半径的阴影贴图算法。其关键核心在于如何制定自适应滤波半径的机制。

### SDF的阴影

有向距离场（Signed Distance Field，简称SDF）表示为空间中的一个点到场景中所有物体表面的最近距离。
基于距离场的软阴影结合了光线步进的思想，其本质上并非物理准确（比PCSS还要更fake一些），但仍然能够产生非常不错的软阴影效果。该方法在shader toy上非常常用。
从需要计算阴影强度的着色点出发，向光源发射一条Shadow Ray，沿着该方向进行光线步进。每步进到一个点，我们可以得到一个圆心为当前点、半径为当前点的SDF值的圆（三维情况下为球体），从出发点向该圆作一条切线，可以得到一个夹角。
取所有这些夹角中的最小值，根据这个值来确定半影大小。

### CSM

### OSM

使用动态浮点立方体贴图来实现向所有方向投射阴影的点光源的阴影效果。立方体贴图每帧更新一次，并存储每个片段到光源的距离，用于确定片段是否有阴影。