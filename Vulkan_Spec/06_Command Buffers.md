# Chapter 6. Command Buffers

命令缓冲区是用于记录命令的对象，这些命令随后可以提交给设备队列执行。
有两个级别的命令缓冲区——主命令缓冲区，可以执行次要命令缓冲区，并提交给队列，次要命令缓冲区可以由主命令缓冲区执行，不能直接提交给队列。

一个已被记录的cmdbuffer有很多种，包括绑定了管道和描述符的命令，修改动态状态的命令，绘制命令等等等等。

每个命令缓冲区独立于其他命令缓冲区管理状态。主命令缓冲区和辅助命令缓冲区之间或辅助命令缓冲区时没有状态继承。当命令缓冲区开始记录时，该命令缓冲区中的所有状态都是未定义的。
当记录辅助命令缓冲器以在主命令缓冲器上执行时，辅助命令缓冲器不从主命令缓冲器继承任何状态，并且在记录执行辅助命令缓冲器命令之后，主命令缓冲器的所有状态都是未定义的。

除非另有规定，并且在没有显式同步的情况下，通过命令缓冲器提交到队列的各种命令可以以相对于彼此的任意顺序和/或同时执行。此外，在没有明确的内存依赖性的情况下，这些命令的内存副作用可能对其他命令不直接可见。在命令缓冲区内以及提交给给定队列的命令缓冲区之间都是如此。

## 生命周期

![Alt](./res/command_buffer_lifecycle.png#pic_center)

**Initial**:
当命令缓冲区分配完成(allocated)，它处于初始状态(Initial)。
某些命令能够将命令缓冲区（或一组命令缓冲区）从任何可执行(Executable)、记录(Recording)或无效(Invalid)状态重置回该状态，或者将其释放。

**Recording**:
使用`vkBeginCommandBuffer`将命令缓冲器的状态从初始状态改变为记录状态。一旦命令缓冲区处于记录状态，就可以使用`vkCmd*`命令记录到命令缓冲区。

**Executable**:
使用`vkEndCommandBuffer`结束命令缓冲区的记录，并将其从记录状态移动到可执行状态。可执行的命令缓冲区可以提交、重置或记录到另一个命令缓冲区。

**Pending**:
队列提交命令缓冲区会将命令缓冲区的状态从可执行状态更改为挂起状态。
当处于挂起状态时，应用程序不得试图以任何方式修改命令缓冲区，因为设备可能正在处理记录到它的命令。
一旦命令缓冲区的执行完成，命令缓冲区将恢复到可执行状态，或者如果它是用`VK_command_buffer_USAGE_ONE_TIME_SUBMIT_BIT`记录的，它将移动到无效状态。
应使用同步命令来检测何时发生这种情况。

**Invalid**:
某些操作，如修改或删除记录到命令缓冲区的命令中使用的资源，会将该命令缓冲区状态转换为无效状态。处于无效状态的命令缓冲区只能重置或释放。

任何在命令缓冲区上操作的给定命令都有自己的要求，即命令缓冲区必须处于什么状态，这些要求在该命令的有效使用约束中有详细说明。
重置(Reset)命令缓冲区是一种丢弃任何以前记录的命令并将命令缓冲区置于初始状态的操作。重置是`vkResetCommandBuffer`或`vkResetCommandPool`的结果，或者是`vkBeginCommandBuffer`的一部分（它还将命令缓冲区置于录制状态）。

辅助命令缓冲区(Secondary command buffers)可以通过`vkCmdExecuteCommands`记录到主命令缓冲区。这部分地将两个命令缓冲区的生命周期联系在一起——如果主缓冲区被提交到队列，则记录到其中的主缓冲区和任何辅助缓冲区都会移动到挂起状态。
一旦主命令的执行完成，记录在其中的任何辅助命令也是如此。在每个命令缓冲区的所有执行完成后，它们都会移动到相应的完成状态。
最后，如果辅助缓冲区移动到无效状态或初始状态，则其记录的所有主缓冲区都将移动到无效态。主设备移动到任何其他状态都不会影响其中记录的辅助设备的状态。

## Command Pools

命令池是不透明的对象，从中分配命令缓冲区内存，并允许实现在多个命令缓冲区之间分摊资源创建的成本。
命令池是外部同步的，这意味着一个命令池不能在多个线程中同时使用。这包括通过记录从池中分配的任何命令缓冲区上的命令来使用，以及分配、释放和重置命令缓冲区或池本身的操作。

大概的创建就不多赘述：

```cpp
typedef struct VkCommandPoolCreateInfo 
{ 
  VkStructureType sType;
  const void* pNext; 
  VkCommandPoolCreateFlags flags;
  uint32_t queueFamilyIndex; 
}VkCommandPoolCreateInfo;

VkResult vkCreateCommandPool( 
  VkDevice device,
  const VkCommandPoolCreateInfo* pCreateInfo,
  const VkAllocationCallbacks* pAllocator, 
  VkCommandPool* pCommandPool);
```

### flags

`flags`是`VkCommandPoolCreateFlagBits`的位掩码，指示池及其分配的命令缓冲区的使用行为:

`VK_COMMAND_POOL_CREATE_TRANIENT_BIT = 0x00000001`指定从池中分配的命令缓冲区将是短暂的，这意味着它们将在相对较短的时间内重置或释放。该标志可以由实现用来控制池内的存储器分配行为。

`VK_COMMAND_POOL_CREATE_RESET T_COMMAND_BUFFER_BIT= 0x00000002`允许从池中分配的任何命令缓冲区单独重置为初始状态；
通过调用`vkResetCommandBuffer`，或在调用`vkBeginCommandBuffer`时通过隐式重置。如果未在池上设置此标志，则不得为从该池分配的任何命令缓冲区调用`vkResetCommandBuffer`。

`VK_COMMAND_POOL_CREATE_PROTECTED_BIT= 0x00000004`指定从池中分配的命令缓冲区是受保护的(protected)命令缓冲区时。

### trim

修剪(trim)命令池会将未使用的内存从命令池回收回系统。从池中分配的命令缓冲区不受该命令的影响。
使用`vkTrimCommandPool`或`vkTrimCommandPoolKHR`对命令缓冲池进行trim:

```cpp
void vkTrimCommandPool( 
  VkDevice device, 
  VkCommandPool commandPool, 
  VkCommandPoolTrimFlags flags
);
```

