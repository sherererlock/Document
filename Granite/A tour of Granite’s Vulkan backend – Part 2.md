# A tour of Granite’s Vulkan backend – Part 2

# The life and death of objects

这是一系列的第二部分，我将在其中探索Granite的Vulkan后端。请参阅第一部分了解简介。在本博客条目中，我们将深入探讨代码，我们将从基础知识开始。本条目的重点将是讨论对象生命周期以及Granite如何处理Vulkan规则，即不能删除GPU正在使用的对象。

## Sample code structure

我将从现在开始引用具体的代码示例。我已经开始了一个包含所有示例的小型代码存储库。请查看README.md以了解如何构建，但您不需要运行示例就能理解我用这些示例想要表达的内容。然而，通过调试器逐步执行代码可能会相当有帮助。

## Sample 01 – Create a Vulkan device

在我们进行任何操作之前，我们必须创建一个VkDevice。从Vulkan的这个方面来看，这相当乏味且充满样板代码，就像任何图形API的设置代码一样。从API设计的角度来看，没有太多要覆盖的内容，但有一些要提及的事项。本部分的示例代码在这里：https://github.com/Themaister/Granite-MicroSamples/blob/master/01_device_creation.cpp

这方面的API相当直截了当。我决定将加载Vulkan加载器库的方式分开，因为这里有两个主要的用例：

1. 用户希望Granite从标准位置加载libvulkan.so/dll/dylib并从那里引导。
2. 用户希望加载已经提供的指向vkGetInstanceProcAddr的函数指针。这实际上是常见情况，因为GLFW会动态加载Vulkan加载器，并且您可以使用GLFW提供的glfwGetInstanceProcAddr来引导自己。volk加载器支持这一点。

要创建实例和设备，我们需要做以下常规步骤：

1. 设置Vulkan调试回调
2. 确定并启用相关扩展
3. 在调试构建中启用Vulkan验证层
4. 找到适当的VkQueues以覆盖图形、异步计算和传输

#### Vulkan::Context and Vulkan::Device


上下文（Context）拥有VkInstance和VkDevice，而Vulkan::Device借用VkDevice并管理从VkDevice创建的大对象。在一个VkDevice上可以有多个Vulkan::Device，但我们最终会共享该设备的VkQueues和全局堆，这是Vulkan的一个非常好的特性，因为它允许前端/后端系统（例如RetroArch/libretro）在API边界之间共享VkDevice而不会出现隐藏的全局状态泄漏问题，这在像GL和D3D11这样的传统API中是一个巨大的问题。

请注意，此示例和本章中的所有其他示例都是完全无头的。不涉及窗口系统集成（WSI）。Vulkan非常好，因为我们不需要创建窗口系统上下文来执行任何GPU工作。

## 02 – Creating objects

在图形API中创建新资源应该非常容易，这里我花了很多时间来提高便利性。在原始的Vulkan中创建图像并向其中上传数据是一项艰巨的工作，因为有许多事情你必须考虑。创建临时缓冲区，复制它们，延迟删除该临时缓冲区，也许我们在传输队列上复制，或者不复制？发出一些信号量来将所有权转移到图形队列，创建图像视图，以及太多需要编写的繁琐细节。只是以可靠的方式创建一个图像就需要几百行代码。幸运的是，这种代码非常容易封装成API。请参阅示例：https://github.com/Themaister/Granite-MicroSamples/blob/master/02_object_creation.cpp，其中我们创建了一个缓冲区和图像。我认为这个API的简单程度已经尽可能地简化了，同时保持了合理的灵活性。

#### Memory management

当我们分配资源时，我们会从Granite的Vulkan堆分配器中分配资源。如果我今天重新设计Granite，我可能会直接使用AMD的Vulkan内存分配器，但是在我设计我的分配器时，它还不存在，而且我对我目前的设计相当满意。也许如果将来我需要碎片整理或者一些非常复杂的内存管理策略，我会重新考虑并使用成熟的库。

为了了解算法的要点，Granite将分配64 MB的块，然后将其分割为32个块。这32个块可以进一步细分为32个更小的块，以此类推，直到256字节的小块。我设计了一个可爱的小算法，使用CTZ操作等有效地从这些块中分配内存。经典的伙伴分配器，但你有32个伙伴。

还有专用分配。我使用VK_KHR_dedicated_allocation来查询是否应该使用单独的vkAllocateMemory分配图像，而不是从堆中分配。在某些架构上分配大型帧缓冲区时，这通常很有用。此外，对于超过64 MB的分配，也会使用专用分配。

#### Memory domains

我做的一个很好的抽象是，与其处理像DEVICE_LOCAL、HOST_VISIBLE和所有可能类型的组合这样的内存类型，我事先声明我喜欢我的缓冲区和图像所在的位置。对于缓冲区，有4种用例：

1. Vulkan::BufferDomain::Device – 必须位于DEVICE_LOCAL_BIT内存中。可以是HOST_VISIBLE，也可以不是（独立GPU与集成GPU）。
2. Vulkan::BufferDomain::Host – 必须是HOST_VISIBLE，最好不是CACHED。用于将数据上传到GPU。
3. Vulkan::BufferDomain::CachedHost – 必须是HOST_VISIBLE和CACHED。如果无法使用缓存，则回退到非缓存状态，但应该不会发生。可能不是COHERENT。用于从GPU读取数据。
4. Vulkan::BufferDomain::LinkedDeviceHost – HOST_VISIBLE和DEVICE_LOCAL。这映射到AMD的固定PCI映射，限制为256 MB。我不认为我曾经真正使用过它，但如果有需要的话，这是一个小众选项。

当将初始数据上传到缓冲区时，如果使用Device选项，我们可以利用与CPU共享内存的集成GPU。在这种情况下，我们可以避免任何临时缓冲区，直接将数据memcpy到新的DEVICE_LOCAL内存中。当你不需要时，不要盲目使用临时缓冲区。集成GPU通常会有DEVICE_LOCAL和HOST_VISIBLE内存类型。

#### Mapping host memory

虽然在示例中没有提到，但讨论如何将Vulkan内存映射到CPU是有意义的。一般而言，一个很好的经验法则是保持主机内存持久地映射。vkMapMemory和vkUnmapMemory非常昂贵，特别是在移动设备上，而且我们一次只能对VkDeviceMemory（64 MB有大量子分配！）进行一次映射。与其一直进行Map/Unmap操作，我们在Vulkan::Device中实现了映射/取消映射，通过检查是否需要执行缓存维护，而不会增加额外的CPU成本。例如，在map()中，如果内存类型不是COHERENT，我们需要调用vkInvalidateMappedRanges，而对于unmap，如果内存不是COHERENT，我们则调用vkFlushMappedRanges。这在移动设备上是相当常见的，特别是在从GPU进行读取时，因为我们需要CACHED，但可能得不到COHERENT。Granite的后端将所有这些都抽象化了。

#### Physical and transient image memory

Vulkan的一个非常强大的特性是支持TRANSIENT图像。这些图像在Vulkan中不必由物理内存支持，这对于基于瓦片的移动渲染器非常有用。

在Granite中，我完全支持TRANSIENT图像，可以为图像传入两种不同的领域，即物理和瞬态。由于TRANSIENT图像通常用于一次性场景，因此在Vulkan::Device::get_transient_attachment()中有一个便利的方法，可以简单地请求一个用于渲染的瞬态图像，包括格式和分辨率。由于瞬态图像在内部管理起来非常容易，通常不会手动创建瞬态图像。

#### Handle types

在一般情况下，有许多抽象处理类型的方法，但我选择了我自己的“智能指针”变体，即可信的侵入式引用计数指针。它基本上可以被视为std::shared_ptr，但更简单，而且我们可以非常好地池化句柄的分配。我们如何设计这些句柄类型对于Vulkan来说并不重要，但我认为这一点可能会引发一些问题，所以我在这里进行了解释。有关详细信息，请参阅https://github.com/Themaister/Granite/blob/master/util/intrusive.hpp。

## 03 – Deferring deletions of GPU resources

现在我们开始涉及到在Vulkan后端之间可能存在重大设计差异的主题。我的设计理念是，在中间级别的抽象中，要方便、确定性，并在牺牲理论上的最佳解决方案的情况下达到足够好的效果。

在Granite中你会发现一个常见的主题是RAII的使用。一旦对象的生命周期结束，我们就会自动清理资源。这对于C++程序员来说并不新鲜，但在Vulkan中的一个大问题是，我们不仅仅是在CPU上管理内存，使用new/delete。实际上，我们需要仔细控制何时删除事物，因为GPU可能正在使用我们正在释放的资源。在这里的策略是延迟任何删除操作。示例在这里：https://github.com/Themaister/Granite-MicroSamples/blob/master/03_frame_contexts.cpp。

#### The frame context

为了处理Granite中的对象生命周期，我引入了一个名为frame context（帧上下文）的概念。帧上下文负责保存属于一个“帧”工作的所有资源。通常，这对应于在AcquireNextImage和QueuePresent之间的工作帧，但它们并不是紧密耦合的。Device有一个帧上下文的数组，通常有2个，以允许CPU和GPU之间的双缓冲（在Android上有3个，因为TBDR GPU更倾向于使用稍多的缓冲，它们的管线更加流水化）。帧上下文基本上是一个巨大的数据结构，其中包含诸如：

- 必须等待的VkFences，以确保与此队列相关的所有GPU工作都已完成。这是我们所有回收和删除的门卫。
- 每个工作线程和队列类型的命令池。
- 一旦Fences发信号就要删除的VkBuffers、VkImages等。
- 要释放的来自堆分配器的内存分配。
- ...以及其他各种资源。

基本上，我们有一个中心位置来存放任何需要在GPU保证完成此帧工作后发生的“后续”事项。

作为特殊考虑，大而且重要的“让它变慢”调用Device::wait_idle()将自动一次性清理所有内容，因为它知道此刻GPU没有执行任何操作。

#### Command buffer lifetime compromise

在实践中使基于帧的清理工作起来，我们需要简化对命令缓冲区可以执行的操作的概念。在Vulkan中，我们有灵活性可以记录命令缓冲区一次，并随时重用它们。这会带来一些复杂性。首先，它使得每帧命令池的概念不再适用。在这种情况下，我们永远不能重置命令池，因为可能会有自由浮动的命令缓冲区，稍后可能会被使用。在Vulkan中，当你不允许释放单独的命令缓冲区时，命令池的效果最佳。

如果我们有可重用的命令缓冲区，我们也会遇到对象生命周期的问题。我们最终会面临一个棘手的情况，即GPU资源必须保留，直到所有引用它们的命令缓冲区都被丢弃。这导致了一个非常困难的情况，你有两个选择——为每个命令缓冲区进行深度引用计数，或者只是祈祷所有这些都能顺利进行，并确保对象保持活动状态直到不再需要为止。前一种选择成本高昂且容易出错，后一种对于我来说就像在用剃刀片玩耍，用户承担了很大、毫无意义的负担。

我一般认为可重用的命令缓冲区并不是一个值得的想法，至少对于交互式应用程序来说不是。我们不会一次又一次地向GPU提交静态工作负载，所以这并没有太多合理的用例。我们可以一次又一次地提交相同的调用的途径可能仅限于后处理，但是记录一些渲染几个全屏四边形的绘图调用（或者对于新潮的孩子们来说是计算调度）并不是您绘图调用开销真正会影响的地方。

我发现初学者对于积极重用的想法过于着迷。最后我觉得这是误导的，而且有许多更好的地方可以花时间。在Vulkan中记录命令缓冲区本身非常高效。

我对命令缓冲区的理想用法是，命令缓冲区是轻量级句柄，所有句柄都线性地来自一个共享的命令池。不重用，因此我们在池上使用ONE_TIME_SUBMIT_BIT和TRANSIENT_BIT。

在Granite中，我极大地简化了命令缓冲区的概念，将其作为临时句柄请求、记录和提交。它们必须在您请求命令缓冲区的同一帧上下文中记录和提交。这样，我们就不需要跟踪每个命令缓冲区的对象了。相反，我们只需要将资源销毁与帧上下文关联起来，就这样。不需要复杂的跟踪，非常高效，但我们有一点风险，即在理论上最佳时机之后稍微延迟销毁对象。在某些情况下，这可能会增加内存压力，但我认为我做出的权衡是正确的。如果需要，我可以随时添加显式的“现在删除这个资源，我知道是安全的”方法，但我还没有发现任何这方面的需求。这只有在我们真正受到内存限制时才重要。

我做出的设计决策是，我永远不想为像图像和缓冲区这样的资源进行内部引用计数，并且设计将被迫不依赖于您通常在遗留API实现中找到的细粒度跟踪。任何引用计数的操作都应立即对API用户可见，永远不会隐藏在API实现背后。实际上，用于绑定资源的命令缓冲区参数是普通的引用或指针，而不是引用计数类型。

#### The memory pressure of very large frames

这种设计的主要缺陷是可能存在一种情况，即存在一个帧上下文，其对资源的创建和删除使用极端频繁。一个典型的例子是加载屏幕或类似的情况。由于Vulkan资源直到帧上下文本身完成才被释放，我们无法在一个帧内回收内存，除非我们明确地使用Device::next_frame_context()迭代帧上下文。这种权衡意味着Granite后端不必通过启发式方法在适当的时候阻塞GPU以回收内存，这会增加很多复杂性，并破坏Granite行为的确定性。

## … up next!

在Granite研究的下一篇文章中，我们将讨论着色器管线，其中包括VkShaderModule、VkDescriptorSetLayout、VkPipelineLayout和VkPipeline对象。