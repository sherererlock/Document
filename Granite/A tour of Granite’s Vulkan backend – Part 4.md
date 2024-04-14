# A tour of Granite’s Vulkan backend – Part 4

# Optimizing for scratch data allocation

从堆中分配内存是可以的，但在引擎中，我们经常需要分配一次性数据。统一缓冲区就是一个完美的使用案例。使用临时命令缓冲区时，某些数据也将是临时的。通常情况下，我们只是想要将一些常量数据发送到绘图调用中，并忘记它。

在Vulkan中，有一个非常适合这种情况的描述符类型，即VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER_DYNAMIC。不仅仅是统一缓冲区，我们经常希望分配临时顶点缓冲区、索引缓冲区和纹理更新的临时数据。

对我来说，在Vulkan中实现这些分配器而没有API开销是一件非常重要的事情。在传统API中，存在非常痛苦的限制，即“快速分配并忘记”缓冲区分配非常难以实现。通常情况下，当提交绘图调用时，缓冲区通常无法映射，因此我们需要努力思考写时复制行为、丢弃行为、调用映射/取消映射的API开销（这会破坏线程驱动程序的优化）或批量分配和在额外的几次内存复制数据之间移动。这太痛苦了，如果我们没有触及所有快速路径，很多CPU性能可能会荡然无存。

我能想到的传统API中的唯一合适解决方案是GL 4.5的GL_ARB_buffer_storage，它支持类似于Vulkan的持久映射缓冲区，但依赖GL 4.5（或GLES 3.2 +扩展）对我来说似乎不太合理，因为GL的目标应该被视为对没有Vulkan驱动程序的旧GPU的兼容选项。这个功能是GL时代“接近零驱动程序开销”（AZDO）的一个基石。对于Windows来说，D3D11仍然将是“兼容”选项，并且忘记依赖Android上最新的和最伟大的GLES。

这是一个完美的机会，可以呈现一个使用大部分这些功能的“你好三角形”示例，但我们首先需要WSI，所以让我们从那里开始。

### 06 – Pushing pixels with SDL2

Granite的主要代码库通常使用GLFW，因此为了使示例更少冗余，我编写了这个示例，使用SDL2的Vulkan支持，这与从SDL 2.0.8开始的GLFW的支持非常相似。

实现WSI类似于实例和设备创建，其中有大量的样板代码要处理，几乎没有设计考虑的余地。Granite的WSI实现有两个主要路径：

#### On-screen / VK_KHR_surface

在这种模式下，WSI类自动为我们创建和拥有Vulkan::Context和Vulkan::Device，并拥有一个VkSwapchainKHR。它无法自行完成的唯一任务是创建VkSurfaceKHR，这取决于平台。幸运的是，表面是唯一依赖于平台的对象，因此当Vulkan::WSI需要时，我们可以提供一个接口实现来创建这个表面。示例实现了一个SDL2Platform类，它使用SDL2的内置包装器来创建表面，非常方便！

#### Off-screen / externally owned swapchains

Granite还用于我们不一定拥有显示在屏幕上的交换链的情况。我们可能希望提供已经创建的图像来替代VkSwapchainKHR，并提供我们自己的图像索引以及获取/释放信号量。完成一帧后，我们可以将虚拟交换链图像传递给我们的消费者。这种情况的主要案例是Granite中的libretro API实现。

#### Pumping the main loop

Vulkan的Acquire/Present模型直接映射到Granite中的“begin”和“end”模型。我们调用Vulkan::WSI::begin_frame()来获取一个新的图像索引，推进帧上下文并处理任何帧间工作。在这里，我们可能需要处理过时的交换链和各种清理工作，这在旧的API中从未考虑过。

对于WSI图像的信号量是自动处理的。WSI图像被特别对待，并且自动处理WSI资源的同步是非常直接的，以至于没有必要将其暴露给用户。（在Granite中，同步通常是非常显式的，但WSI是少数例外之一。）主循环大致如下：

```
wsi.begin_frame(); // <-- acquire image if necessary, advances frame context
auto cmd = device.request_command_buffer();
// do work and render to swapchain
device.submit(cmd);
wsi.end_frame(); // <-- flushes frame, queues up presents if swapchain was rendered to this frame
```

总的来说，在Vulkan中抽象WSI代码是必要的，我对最终实现的灵活性和简单易用性感到满意。

### 07 – Hello triangle (quad?) with scratch allocated VBO, IBO and UBO

现在我们可以在屏幕上显示东西了，现在我们开始进入这篇文章的实质内容。https://github.com/Themaister/Granite-MicroSamples/blob/master/07_linear_allocators.cpp 在WSI示例中增加了一个漂亮的小矩形。顶点缓冲对象（VBO）、索引缓冲对象（IBO）和统一缓冲对象（UBO）直接在命令缓冲区上分配。

![img](http://themaister.net/blog/wp-content/uploads/2019/04/Screenshot-from-2019-04-23-15-59-12.png)

#### Linear allocator – allocating memory at the speed of light

这种分配器有很多名称——链式分配器、增量分配器、临时分配器、堆栈分配器等。这是当我们想要分配大量小块内存，并在将来的某个时刻全部清除时的完美分配器。分配是通过增加偏移量来完成的，释放是通过将偏移量重新设置为0来完成的，即一次性将所有内存都“清除”。

#### Buffer pools of linear allocators

某些引擎实现的策略是只有一个巨大的线性分配器在运行，一旦耗尽，就被视为内存耗尽，GPU 将不可避免地停滞。从“显式描述符集”设计的角度来看，如果使用 UNIFORM_DYNAMIC 描述符类型，这种策略是不错的，因为我们可以使用固定的描述符集来存储统一数据，当绑定描述符集时，会对 UBO 中的偏移量进行编码。我觉得这个概念有点过于限制性，因为使用情况没有明显的限制（非常依赖内容和场景）。相比之下，我选择了一个由较小缓冲区组成的循环池，因为 Granite 的描述符绑定模型非常灵活，正如我们在本系列的上一篇文章中所看到的。如果必须处理显式描述符集，统一数据将会很难处理。

Vulkan::CommandBuffer 可以从 Vulkan::Device 请求适当大小的数据块，一旦耗尽或提交，缓冲区就会被再次回收。我们只能在 GPU 上的帧完成后重新使用缓冲区，因此我们还使用帧上下文在正确的时间将线性分配器回收到“准备好分配”的池中。

#### To DMA queue or not to DMA queue …

离散显卡具有一种特性，即在 VRAM 中访问内存非常快，而主机内存可以通过 PCI-e 以更慢的速度访问。对于像顶点、索引和统一缓冲区这样的暂存数据，假设我们应该将 CPU 端的数据复制到 GPU 端，让 GPU 在快速的 VRAM 中消耗流数据可能是合理的。Granite 支持两种模式，一种是让 GPU 从 HOST_VISIBLE 中只读数据，另一种是我们自动执行从 CPU 缓冲区到 GPU 的暂存缓冲区复制。

然而，在实践中，我没有看到进行暂存复制带来的任何收益。提交一个命令缓冲区到 DMA 队列以复制数据的额外开销，以及添加额外的信号量和其他同步开销似乎都不值得。离散显卡可以很好地缓存来自 PCI-e 的只读数据。

#### Super-convenient API

由于我们有一个非常自由流畅的描述符绑定模型，我们可以拥有这样的 API：

```
auto cmd = device.request_command_buffer();
MyUBO *ubo = cmd->allocate_typed_constant_data<MyUBO>(set, binding, count);
// Fill in data on persistently allocated buffer.
ubo->data1 = 1;
ubo->data2 = 2;

void *vert_data = cmd->allocate_vertex_data(binding, size, stride);
// Fill in data.
void *index_data = cmd->allocate_index_data(size, VK_INDEX_TYPE_UINT16);
// Fill in data.
cmd->draw_indexed();

// Pointers are now invalidated.
device.submit(cmd);
```

分配函数只是轻量级包装，用于分配并在适当的偏移量处绑定缓冲区。完全可以自行编写线性分配系统，例如，您希望在同一帧中的多个命令缓冲区中重用丢弃的分配，或者类似的情况。

### Conclusion

我认为花时间让临时分配尽可能方便会带来巨大的回报。知道你可以在命令缓冲区中分配数据，几乎没有额外开销，简化了调用点周围的大量代码，而且实现这一点几乎没有成本。线性分配器的实现非常简单。

### … up next!

在下一集的“这一切看起来都很高级，我在哪里可以找到低级的东西”中，我们将看看Granite中的渲染通道和同步，这是Granite暴露出的低级方面。