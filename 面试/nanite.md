# Nanite UE

## 目标和意义

Nanite部分的管线与deferred兼容

### 缺陷和局限性

不适合简单模型，很多复杂的逻辑可能会构成负优化
目前不支持同一母材质不同材质示例之间的合并渲染（shading阶段）
目前不支持半透透明物件（alpha blend类型的，alpha test后续可能会支持）的渲染
目前只支持刚体模型，不支持蒙皮等会导致模型形变的模型
当某个cluster的某个面片距离最上层surface非常近的话，可能无法被culled掉，从而导致overdraw增加，这种情况对于远距离的物件来说比较常见。

### 经典渲染管线

* Command Buffers：这是渲染管线的输入部分，用于存放渲染指令。这些指令可以是绘制（Draw）命令，也包含其他与渲染流程有关的命令。开发者会通过编程来填充命令缓冲区，告诉GPU接下来需要进行哪些操作。
* Command Processor：处理命令缓冲区中的命令，并将它们转发到管线的下一个阶段。这一阶段负责解码指令并且优化指令流以供后续处理。
* Graphics Processor：是指整个负责处理图形任务的硬件部分，包括后面提到的光栅化器、计算单元等。
* Rasterizer：光栅化是将图形从几何描述（如顶点数据）转换为像素数据的过程。这个阶段确定了最终图像中哪些像素应当被涂上颜色，并且这些颜色是什么。
* Compute Units：计算单元是处理图形计算任务的核心。这些单元可以并行处理多个任务，比如着色、贴图处理、几何计算等。
* Render Backend：渲染后端通常包含深度测试、混合、反锯齿等操作，它是渲染管线的最后阶段，负责将处理好的像素数据最终输出到帧缓冲区，从而显示在屏幕上。

### 间接绘制

允许从GPU缓冲区中读取参数来执行绘制命令，而不是直接从CPU传递参数。这样做的好处是可以减少CPU到GPU的命令流量，并允许GPU更自主地管理绘制过程。其允许合并多个绘制调用到单个间接绘制命令中，即使是使用不同的网格拓扑结构。

最理想的状态：GPU-Driven渲染管线
GPU控制实际渲染的对象，包括级别细节（LOD）的选择和可见性剔除（visibility culling）。
不存在CPU到GPU的往返交互，这意味着CPU不需要处理GPU数据。
支持多个视口/视椎体（frustums）。

### 大致流程

* 计算通道（Compute pass）：GPU执行视锥剔除（Frustum culling）和遮挡剔除（Occlusion culling）等计算任务，这些任务确定哪些对象可见，哪些应被渲染。
* 编码遮挡器绘制（Encode occluder draws）：生成对于遮挡对象的绘制命令。
* 渲染遮挡器（Render occluders）：实际渲染确定为遮挡物的对象。
* 生成遮挡器数据（Generate occluder data）：生成后续渲染步骤可能需要的遮挡信息。
* 再次计算通道：执行更多的剔除和级别细节选择。
* 编码绘制（Encode draws）：生成场景中所有对象的绘制命令。
* 渲染通道（Render pass）：最终渲染整个场景。

## 实现原理

### 预处理

对一个模型，将Meh中每相互临近的128个Triangle组合在一个结构，称之为Cluster，128的设计是为了迎合Memory Cache。将8～32个Cluster组合成一个组，叫做Cluster Group。
对于原始未进行几何优化的部分，称为LOD0.

划分Cluster后，当进行LOD的切换后，切换的便不再是整个模型，而是局部的Cluster Group。

### LOD生成

模型LOD的生成可不能像Texture那样简单了，新生成的LOD并不会完全包含上一级LOD的所有Cluster，边界都会完全不一样。也就是说，每一个低级的Cluster可能会影响到数个上一层的LOD下的Cluster。
具体来说，就是对于Cluster内部的Mesh进行简化，通过Quadric Error Matrics算法进行简化，而参与计算的Mesh自然就是Cluster Group内三角形的集合。
基于二次误差作为度量代价的边收缩算法（Quadric Error Matrics）：对于网格中的每一个顶点v，预定义一个4x4的对称误差矩阵Q。对于一条边的收缩前后的不同向量，选择一个使收缩代价最小的位置。或者计算顶点位置使得最小，通过计算一阶导数为0的方程求解。

通过使用Cluster Group代替Cluster避免边缘的面数密度过高，通过变换边缘规避边缘永恒不变的视觉误差。

### 运行时剔除

为了准备运行时的剔除工作，通过BVH组织Cluster Group，BVH中的Bounding Box以Cluster Group为基础进行构建，也就是使用Group内的Mesh计算Bounding Box大小。从最小粒度上来说，每个叶子结点就是一个Cluster Group。

### 运行时CPU阶段

* 粗略视锥剔除（Coarse Frustum Culling）: 这是一个剔除阶段，目的是快速排除那些不在摄像机视锥内的物体，这样它们就不会进入后续的渲染流程。这一步骤通常**在CPU上执行**，因为它涉及空间数据结构的遍历和逻辑决策，这些任务CPU擅长处理。
* 构建批次哈希（Build Batch Hash） / 更新实例GPU数据（Update Instance GPU Data） / **批量绘制调用**（Batch DrawCalls）: 这一系列步骤涉及优化渲染调用和更新需要在GPU上渲染的对象的数据。目的是为了减少绘制调用的数量并确保数据在GPU上是最新的。
* 实例剔除（Instance Culling）: 更进一步地**剔除**不可见的实例，这可能包括因遮挡（物体被其他物体挡住）或背面（背对摄像机的面）而不可见的实例。
* 集群块扩展（Cluster Chunk Expansion） / 集群剔除（**Cluster Culling**）: 在GPU上进行的这些步骤进一步提高剔除效率，将场景划分为更小的集群，以更精细的粒度进行剔除。
* 索引缓冲压缩（Index Buffer Compaction）: 这一步优化了内存使用和绘制调用，通过**压缩索引缓冲区**以减少不必要的内存使用，并加速绘制过程。
* 多重绘制（Multi-Draw）: 最后，GPU执行实际的渲制调用，利用它的并行计算能力绘制所有剩余的可见物体。

具体的对于CPU来说，其在初始阶段（1和2阶段）负责非常简单的剔除和数据准备，重点在于让GPU更好地进行剔除和渲染：

* 非常粗略的视锥剔除（Very Coarse Frustum Culling）: 这一步骤是一个关键优化。它利用四叉树来快速确定哪些物体位于摄像机的视锥内，哪些不在。
* 基于材质的批次处理（Batching by Material）: 一旦决定了哪些物体是可见的，下一步是将具有相同材质的物体分批处理。这样做可以减少状态改变（比如切换不同的纹理或着色器程序），这是图形渲染中开销最大的操作之一。
* 根据哈希合并绘制调用（Drawcalls Merged Based on Hash）: 这个过程涉及创建一个哈希表，这个表基于非实例化数据（例如材质和渲染状态）。绘制调用（Drawcalls）是指令CPU告诉GPU渲染一组图形对象。通过合并具有相同哈希值（即相同非实例化数据）的绘制调用，可以进一步优化渲染过程。
* 更新每个实例的数据（Update Per Instance Data）: 对于每个实例（即场景中的每个独立对象），需要更新它们的变换矩阵（位置、旋转、缩放），LOD等信息。

CPU处理后提交给GPU端的数据叫做Nanite Mesh。
Nanite Mesh会先用CS做剔除，剔除后剩下的Mesh会检测其占屏面积，如果符合Small Triangle的规格，则送入软光栅化的管线，否则还是走硬件光栅化的流程。
但不管走哪条管线，最终都会输出一个叫做Visibility Buffer的东西。Visibility Buffer在之后会被转换成**GBuffer**。

### 运行时GPU阶段

* Instancing Culling
剔除，也就是Culling是Nanite Mesh管线的第一步，首先是**Instancing Culling**，其以整个**Mesh**为单位进行剔除，主要是基于视锥和上一帧生成的Hi-z缓冲进行剔除。

* Hierarchical/Persistent Culling
使用提前构建的BVH来实现对**Cluster Group**对剔除，同样也使用了视锥剔除和Hi-z剔除两种方式。
在这一阶段也会完成LOD的选择，以Cluster Group为单位进行遍历，将相机和Group的距离转为Error Threshold，然后比对每个Group自身的Self Error和Parent Error

* Cluster Culling
与对Cluster Group的剔除一样，Cluster也同样需要做一遍视锥剔除和Hi-z剔除。
不过除此之后还有项增加功能便是标记Cluster为Large Triangle还是Small Triangle，这个标记是针对整个Cluster而非单个Triangle。因为最终是把每个Cluster送入管线而非每个Triangle。

以上三个阶段的剔除，粒度分别从大到小。所有的剔除工作和LOD都是通过GPU的多线程实现并行处理的。UE通过一个MPMC队列完成任务的消费。

#### 关于剔除的题外话

背面剔除（backface culling）包含几个过程

1. 对于每个像素级的视锥，预先计算并存储场景内每个三角形的可见性信息
2. 基于摄像机的方向，将一个Cluster当成一个cubemap，以该摄像机射出的射线和其相交的面获取可见性信息。
3. 系统会获取一个64位的数据，这64位代表了该Cluster中所有三角形的可见性。每一位代表一个或多个三角形，1代表可见，0代表不可见。

刺客信条大革命使用了遮挡深度生成（Occlusion Depth Generation）技术，免于渲染而获取遮挡信息，这狗几把游戏真是白瞎了这么好的技术。

1. 基于场景深度的预处理（Early-z和Hi-z），用全分辨率确定最好的遮挡体（occluders），这些遮挡体可以由艺术家手工选择，或者通过启发式算法自动选择。
2. 挖掘遮挡体导致的问题（Holes from rejected occluder）: 如果选择了错误的遮挡体，或者是几何形状经过Alpha测试后被排除，可能会在深度图中留下“洞”。
3. 下采样（Downsampling）: 为了效率，选出的最好的遮挡体的深度信息被下采样到更低的分辨率，比如512x256。
4. 将低分辨率下的深度图和全分辨下的深度图进行结合，也是用于填补因遮挡体选择不当或者移动物体造成的洞。
5. 处理误遮挡。
6. 生成一个Hi-Z缓冲区。

### 光栅化

硬光栅化在处理小面片上会有较大程度的性能浪费，因此这里将这一部分面片使用CS来进行软光栅化（并不是在CPU阶段进行）。

软光栅化采用了常规的扫描线算法，每个Cluster启动一个CS
具体流程：

1. 读取Index Buffer并计算出Cluster内三角形顶点的Clip Space Vertex Positon
2. 根据Clip Space Vertex Position计算出三角形的边，并执行背面剔除和小三角形（小于一个像素）剔除。
3. 缓存三角形的Clip Space Vertex Positon到shared memory。
4. 使用扫描线算法对三角形进行光栅化，并利用原子操作完成Z-Test，最终将数据写进Visibility Buffer

Visibility Buffer实际为一张R32G32_UINT的贴图(8 Bytes/Pixel)，其中R通道的0~6 bit存储Triangle ID，7~31 bit存储Cluster ID，G通道存储32 bit深度。（把深度放在最高的32位其实是利用的显卡的一些特性，在使用R32G32_UINT时，它会自动比对高32位的数值确保只写入最大数值）

将Visibility Buffer转换为Gbuffer