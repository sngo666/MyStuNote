# Chapter 7. Synchronization and Cache Control

在Vulkan中，同步访问资源主要是应用程序的责任。相对于主机和设备上的其他命令，命令的执行顺序几乎没有隐含的保证，需要显式指定。内存缓存和其他优化也是显式管理的，要求通过系统的数据流在很大程度上受应用程序控制。
Vulkan提供了五种显式同步机制：

**Fence**: Fnece可以用来通知主机设备上某些任务的执行已经完成，控制主机和设备之间的资源访问。
**Semaphores**: Semaphores用于控制跨多个队列的资源访问。
**Events**: Events提供了一个细粒度的同步原语，它可以在命令缓冲区中发出信号，也可以由主机发出信号，并且可以在命令缓冲区中等待，也可以在主机上查询。事件可用于控制单个队列中的资源访问。
**Pipeline barriers**: barriers在单个命令缓冲区内提供同步控制，但在单个点上，而不是使用单独的信号和等待操作。管道屏障可用于控制单个队列内的资源访问。
**Render pass objects**: dependency为渲染任务提供一个同步框架，建立在本章的概念之上。许多需要应用程序使用其他同步原语的情况可以更有效地表示为呈现通道的一部分。渲染通道对象可用于控制单个队列中的资源访问。

## 执行和内存依赖

**operation**是要在主机、设备或外部实体(如presentation engine)上执行的任意数量的工作。同步命令在命令的两个同步作用域定义的两组操作之间引入显式的执行依赖关系和内存依赖关系。
同步范围(synchronization scopes)定义了同步命令能够与哪些其他操作创建执行依赖关系。不在同步命令的同步作用域中的任何类型的操作都不会包含在生成的依赖项中。例如，对于许多同步命令，可以将同步范围限制为仅在特定管道阶段执行的操作，而将其他管道阶段排除在dependency之外。

**execution dependency**是对两组操作的保证，第一组操作必须发生在第二组操作之前。如果一个操作发生在另一个操作之前，那么第一个操作必须在第二个操作初始化之前完成。

假设$Ops_1$和$Ops_2$是完全独立的操作，并定义$Sync$为同步命令
