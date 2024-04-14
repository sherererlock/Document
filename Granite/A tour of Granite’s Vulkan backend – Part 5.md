# A tour of Granite’s Vulkan backend – Part 5

# Render passes and synchronization

这是Granite Vulkan后端导览的第5部分。我们将深入探讨我认为是学习 Vulkan 最困难的方面，掌握这些 Vulkan 主题是真正掌握的难点。这种理解水平是高级API无法达到的。

本文不旨在成为Vulkan同步的教程，因此我会假设读者具有一定的基础知识。

### Render passes

渲染通道是Vulkan中的一个全新基础部分，在任何旧的API中都不存在。在旧的API中，你可以随心所欲地渲染到帧缓冲区，但这种方法在基于瓦片的GPU上总是很糟糕，而且现在在混合瓦片架构上，对于桌面系统来说可能也很糟糕。Vulkan 强制你考虑一次性将所有需要渲染的内容渲染到一个帧缓冲区，然后再进行下一个。

在Granite中，我希望确保大部分 Vulkan 渲染通道的灵活性和显式性可以用最少的样板代码表达出来。大多数项目似乎不太关注这一部分，除了它是你必须做的事情之外，并且很少有人看到它们带来的好处。如果你不关心移动端性能，那么这可能是一个合理的立场。但如果你需要针对移动端，额外的工作是值得的。截至目前，这个特性非常侧重于移动端，但桌面GPU似乎正在逐渐向基于瓦片的架构迈进，因此很有意思的是看看这种对渲染通道的观点是否会在未来发生变化。即使最近的D3D12也加入了渲染通道，尽管是以简化的形式。

在最基本的形式中，Vulkan 中的渲染通道可能是相当令人生畏的设置，这是你必须做的许多战斗之一，以在屏幕上显示简单的三角形。如果我们考虑一个只有一个子通道的渲染通道（99%的情况下我们关心的情况），我们需要事先指定：

- 有多少个附件？
- 使用哪些格式？
- 有多少个MSAA样本？
- initialLayout 和 finalLayout
- 渲染时使用哪种图像布局？
- 我们是从内存加载还是在渲染通道开始时清除附件？
- 我们是否要将附件存储到内存中？
大部分这些信息都是我们可以自动化的样板代码，但像加载/清除/存储这样的操作在后端之前我们无法推断。事先知道这种信息对带宽消耗可能非常有益，至少在移动端是如此。

丑陋的帧缓冲对象
Vulkan 的一个丑陋之处是使用 VkFramebuffer。我想要一个API，只需说“开始一个渲染通道，我们将渲染到这些附件”。在GL中预先创建“FBOs”真的很丑陋，我认为将API用户拖着拥有代表几乎没有用处的对象所有权是一个糟糕的抽象。FBOs 是空的外壳，实际上可能只是一个图像视图数组。

我们可以在每次开始渲染通道时创建 VkFramebuffers，然后立即推迟删除它，但是创建这些对象是有一些成本的。驱动程序中至少需要一个句柄分配，而在某些驱动程序中可能需要更多。在这里，我只是重用了我在描述符集模型帖子中介绍的临时哈希图分配器。VkFramebuffer 对象可以在多个帧中重复使用，但如果它们一段时间没有被使用，它们就会被删除，因为 VkFramebuffer 对象是不可变的。

#### Automating VkRenderPass creation

这个主题实际上在我们开始深入研究 Vulkan 渲染通道时会变得相当复杂，但我们可以从简单的情况开始。Granite 中的 API 大致如下：

```
Vulkan::RenderPassInfo rp;
rp.num_color_attachments = 1;
rp.color_attachments[0] = &rt->get_view();
rp.store_attachments = 1 << 0;
rp.clear_attachments = 1 << 0;

rp.clear_color[0].float32[0] = 1.0f;
rp.clear_color[0].float32[1] = 0.0f;
rp.clear_color[0].float32[2] = 1.0f;
rp.clear_color[0].float32[3] = 0.0f;

cmd->begin_render_pass(rp);
```

这是立即开始渲染通道的一种方式，没有必要预先创建帧缓冲等。VkRenderPass 可以像其他所有内容一样按需延迟创建。

#### Formats / MSAA sample counts

渲染通道需要知道格式和样本计数，由于我们直接在 begin_render_pass() 中传递具体附件，因此我们在这里就有了所需的信息。

#### Image layouts and VK_SUBPASS_EXTERNAL dependencies

在Granite中有三种类型的附件：

1. 用户创建的。这些附件是由Device::create_image()创建的渲染目标，后端不拥有资源，也不知道资源将存在多长时间。这是用户创建的渲染目标的常见情况。
2. WSI图像。这些图像很特殊，因为它们来自VkSwapchainKHR或某种等效机制。我们知道这些图像仅用于渲染，并且仅由呈现引擎或其他机制使用。
3. 临时图像。具有临时使用标志的图像仅在渲染通道内部存在。它们不能被采样，它们的内存不一定存在，除非存在指向/dev/null的页表。一旦渲染通道完成，我们不关心这些图像会发生什么。

要推断渲染通道的图像布局，我们有几种不同的方法。

##### WSI IMAGES

我不关心在多个帧中保留WSI图像，并且我也不关心在渲染后从WSI图像进行采样或任何其他奇怪的用例，因此图像布局的流程如下：

- initialLayout = UNDEFINED（丢弃）
- VkAttachmentReference -> COLOR_ATTACHMENT_OPTIMAL或子通道所需的其他布局
- finalLayout = PRESENT_SRC_KHR或在使用外部WSI时所需的其他布局。对于诸如libretro之类的情况，这将是SHADER_READ_ONLY_OPTIMAL，因为图像将被传递给我们不知道或不关心的其他渲染通道。对于无头PNG/视频转储，它可能是TRANSFER_SRC_OPTIMAL。

当initialLayout != 第一个子通道中使用的布局时，vkCmdBeginRenderPass实际上需要隐式执行内存屏障以使其正常工作。重要的问题是这个内存屏障何时发生，答案是“尽可能快”（TOP_OF_PIPE_BIT），如果我们没有在任何地方指定它的话。对于这种情况，Granite将添加一个子通道依赖项，它在COLOR_ATTACHMENT_OUTPUT阶段等待VK_SUBPASS_EXTERNAL。这会与WSI获取信号量保持一致，稍后会详细介绍。

最终的布局 != 最后的布局被使用，因此我们在渲染通道结束时会进行转换，但我们在这里不需要关心外部子通道的依赖关系。自动生成的依赖项是完美的，而且我们将使用WSI释放信号量来适当地同步该图像。

当我们在渲染通道中看到WSI图像时，我们可以轻松地将此命令缓冲区标记为“触摸WSI”。这将影响稍后的命令缓冲区提交。这确实是我之前在早期帖子中提倡的追踪方式，但在这种情况下，它是如此微不足道，以至于我毫不犹豫。

##### TRANSIENT IMAGES

对于临时图像，我们自动化的方式与WSI图像类似，不同之处在于finalLayout将与渲染通道中最后使用的布局相匹配。这样，我们就避免了在渲染通道结束时进行一些无用的图像布局转换。下次我们使用该图像时，它将被丢弃。

因为我们自动处理转换，用户可以自由地从Vulkan::Device中获取临时附件（transient attachment），对其进行渲染，然后忘记它。这对于诸如MSAA渲染之类的事情非常方便，因为多采样附件只需要临时存在，以便被解析成我们关心的单采样附件。我认为必须关心我们不拥有的资源的同步问题很奇怪。

##### OTHER IMAGES

对于任何其他图像，我们需要避免任何隐式布局转换，因此我们简单地强制initialLayout与渲染通道中的第一次使用相匹配，而finalLayout将匹配最后一次使用。在我们的小样本中，这一切都将是COLOR_ATTACHMENT_OPTIMAL。API用户需要知道渲染通道将期望的布局是什么，但将渲染通道映射到期望的布局是很直接的。颜色附件是COLOR_ATTACHMENT_OPTIMAL，深度模板是根据深度的读/写模式为DEPTH_STENCIL_OPTIMAL或DEPTH_STENCIL_READ_ONLY_OPTIMAL，输入附件是SHADER_READ_ONLY，等等。在子通道中，可以将一个附件用于多个目的，Granite也支持这样做。一些示例：

- 颜色附件 + 输入附件：类似于GL_ARB_texture_barrier的反馈循环（对于某些模拟器非常有用）-> GENERAL
- 可读写深度附件 + 输入附件（某些奇怪的贴花算法？）-> GENERAL
- 只读深度 + 输入附件（延迟着色使用案例）-> DEPTH_STENCIL_READ_ONLY_OPTIMAL

所有这些在创建新观察到的VkRenderPass时进行分析，并且子通道依赖关系会自动设置。在渲染通道之外发生的任何事情都是用户的责任。

#### 08 – Bare-bones “deferred rendering” sample

我制作了一个简化的示例来展示API如何在延迟渲染的上下文中表达多通道。其中关键部分如下：

```
Vulkan::RenderPassInfo rp;
rp.num_color_attachments = 3;
rp.color_attachments[0] = &device.get_swapchain_view();

rp.color_attachments[1] = &device.get_transient_attachment(
		device.get_swapchain_view().get_image().get_width(),
		device.get_swapchain_view().get_image().get_height(),
		VK_FORMAT_R8G8B8A8_UNORM,
		0);
rp.color_attachments[2] = &device.get_transient_attachment(
		device.get_swapchain_view().get_image().get_width(),
		device.get_swapchain_view().get_image().get_height(),
		VK_FORMAT_R8G8B8A8_UNORM,
		1);

rp.depth_stencil = &device.get_transient_attachment(
		device.get_swapchain_view().get_image().get_width(),
		device.get_swapchain_view().get_image().get_height(),
		device.get_default_depth_format());

rp.store_attachments = 1 << 0;
rp.clear_attachments = (1 << 0) | (1 << 1) | (1 << 2);
rp.op_flags = Vulkan::RENDER_PASS_OP_CLEAR_DEPTH_STENCIL_BIT;

Vulkan::RenderPassInfo::Subpass subpasses[2];
rp.num_subpasses = 2;
rp.subpasses = subpasses;

rp.clear_depth_stencil.depth = 1.0f;

subpasses[0].num_color_attachments = 2;
subpasses[0].color_attachments[0] = 1;
subpasses[0].color_attachments[1] = 2;

subpasses[0].depth_stencil_mode = Vulkan::RenderPassInfo::DepthStencil::ReadWrite;

subpasses[1].num_color_attachments = 1;
subpasses[1].color_attachments[0] = 0;
subpasses[1].num_input_attachments = 3;
subpasses[1].input_attachments[0] = 1;
subpasses[1].input_attachments[1] = 2;
subpasses[1].input_attachments[2] = 3;  // Depth attachment
subpasses[1].depth_stencil_mode = Vulkan::RenderPassInfo::DepthStencil::ReadOnly;

cmd->begin_render_pass(rp);
// Do work
cmd->next_subpass();
// Do work
cmd->end_render_pass();
```

请参阅示例中的代码注释以获取更多详细信息。在原始的Vulkan中编写这种类型的示例几乎需要一整天的时间。

### Synchronization

Granite的许多方面都相当高级，但在Granite中，同步几乎是完全显式的。Granite的一般理念是，过度追踪资源使用是不可取的，除非这样做非常简单（例如，WSI图像）。同步是一个需要大量追踪才能做好的情况，而且如果要在Vulkan之上实现完全自动的同步，这几乎是不可能的。GL和D3D11驱动程序在这方面有优势，因为它们可以利用GPU特定的同步机制，这些机制可能更适合隐式同步。一个很好的例子是Linux DRM堆栈中的i915驱动程序堆栈，它可以在内核空间中自动同步。我相信这大大简化了Mesa GL驱动程序，但我不知道具体细节。

让我们通过一次思想实验来看看，如果要实现完全自动的屏障系统，我们将遇到哪些大问题。（我尝试过 :p）

#### Problem #1 – We cannot rewind time

当涉及到资源时，我们必须问自己：“这个资源上次是在什么时候和什么地方被访问的？”我们有三种潜在的解决方案来解决隐患：

1. 管道屏障（刚刚被使用过）
2. 事件（在此队列中之前被使用过）
3. 信号量（在不同的队列中被使用过）

理想情况下，我们需要在资源上次使用的确切点注入一个屏障，但我们不能在已经记录的命令缓冲区中间注入新的命令。

在这场斗争中没有赢家，我们要么热切地在每个命令之后注入屏障，希望将来的某个命令需要与之同步（在这里VkEvent非常好用），要么注入屏障太晚，毫无必要地阻塞GPU。

如果考虑到资源将来可能在不同的队列上使用，那么热切地注入屏障就是纯粹的疯狂。这意味着在每个渲染通道或命令之后都要发出沉重的信号量。我们可以简单地不支持多队列，但这是一个巨大的妥协。

#### Problem #2 – Redundant tracking of read-only resources

在尝试实现自动屏障跟踪时我遇到的一个问题是，静态资源可能会在将来被写入，因此我们需要跟踪它们。这是一种浪费 CPU 时间的做法，但可能可以将这些资源明确标记为“不要跟踪，它们确实是静态的，保证不会变动”，但我觉得这是在进行一些不必要的补丁式修复。

#### Problem #3 – Multi-threading

在多线程场景中，“这个资源最后是在哪里被操作的”这个问题实际上可能无法回答。如果我们并行记录命令缓冲区，那么后端在执行 vkQueueSubmit 之前无法了解发生了什么。我见过的解决这个问题的常见方法是，在记录命令缓冲区时在内部解决同步问题，在命令缓冲区提交时，我们可以查看所有使用的资源，并在提交到 vkQueueSubmit 之前解决任何跨命令缓冲区的同步点。然而，复杂性开始急剧上升。这是我们需要重新思考的一个很好的迹象。

我认为当你别无选择，只能将一些旧的遗留 API 移植到 Vulkan 时，你最终会得到这种解决方案，而破坏抽象 API 并不是一个选项。

#### Render graphs

在 Vulkan 后端中解决同步问题只能回顾过去，并在最后一刻处理隐患，但这仅因为我们对应用程序的操作没有任何上下文。在更高层次上，我们可以知道未来会发生什么，并且我们可以在那个层次上自动做出决策，因为我们实际上了解正在发生的情况。这是我不希望在 Vulkan 后端中实现自动同步的另一个原因。要么我们得到一个次优的解决方案，要么我们试图通过启发式方法来填补差距，但现在运行时行为完全不确定。我们应该追求更好的解决方案。

我认为同步问题应该在更高层次上解决，比如渲染图。我一段时间前写过一个关于这个主题的博客：http://themaister.net/blog/2017/08/15/render-graphs-and-vulkan-a-deep-dive/

#### Signalling fences

Granite’s way of signalling fences is very similar to plain Vulkan

```
Vulkan::Fence fence;
device.submit(cmd, &fence);

// Somewhere else, potentially in a different thread.
fence->wait();
// fence ref-count goes to 0, queued up for recycling.
```

有一个 VkFence 对象池，可以重用。对一个 Fence 进行信号通知会强制执行 vkQueueSubmit。一旦 Vulkan::Fence 的生命周期结束，它就会被回收到帧上下文中。这里没有什么特别的。

#### Semaphores

信号量的工作方式与栅栏非常相似，并且在 Device::submit() 中像栅栏一样请求。与Vulkan一样，它们有一个限制，即它们只能等待一次。信号量可以在其他队列中等待，使用 Device::add_wait_semaphore() 在特定队列和管线阶段等待。这与 pDstWaitStages 相匹配。信号量也像栅栏一样被回收利用。

#### Events

事件可以在同一队列中被触发，并且稍后等待。同样，我们有一个 VkEvent 对象池，CommandBuffer::signal_event() 请求一个新的事件，用 vkCmdSetEvent 发出信号，然后将其交给用户。VkEvents 使用帧上下文进行回收利用。还有一个类似的 CommandBuffer::wait_event()，它与 vkCmdWaitEvents 一一对应。

#### Barriers

Granite有许多不同的方法来注入管线障碍，其中最常见的方法包括：

```
cmd->barrier(srcStage, srcAccess, dstStage, dstAccess);
```

这对应于使用VkMemoryBarrier进行的vkCmdPipelineBarrier，以及作用于所有子资源的图像屏障：

```
cmd->image_barrier(image, oldLayout, newLayout, srcStage, srcAccess, dstStage, dstAccess);
```

有些情况下，我们希望批量处理屏障或者使用比这更复杂的命令，因此也有直接传递完整结构的vkCmdPipelineBarrier的接口，但这些接口实际上只在渲染图中使用，因为编写完整结构非常繁琐。

#### The automatic barriers in Granite

有几种情况下我认为自动屏障是合理的。这些情况下很方便这样做，并且不需要进行跟踪，因此我们可以立即解决所有隐患并忘记它。其中一些情况我们已经见过，比如WSI图像和渲染通道中的临时图像。

另一个主要情况是静态只读资源。如果在Device::create_buffer()或Device::create_image()中传递了initial_data，我们通常希望上传一些数据，然后再也不用再去操作它。

总体思路是我们可以使用传输队列通过分段缓冲区上传数据，并注入信号量来阻止所有可能的管线阶段（基于bufferUsage/imageUsage标志）。缺点是如果我们想要一次性上传大量缓冲区或图像，可能会创建太多的提交，但我们可以通过简单地不传递initial_data来选择退出这种自动行为，并自己完成所有的批处理和同步工作。

最终目标是我们应该能够调用create_buffer或create_image并立即使用静态资源，而无需考虑同步问题。

#### 09 – Rendering to image and reading it back to CPU on transfer queue

https://github.com/Themaister/Granite-MicroSamples/blob/master/09_synchronization.cpp

我编写了一个示例，展示了大部分同步API的灵活运用。它在图形队列上渲染一个小的4×4纹理，通过信号量将其与传输队列进行同步，然后将其读取回一个CACHED主机缓冲区。我们生成了等待在栅栏上的线程，映射缓冲区并读取结果。

### Conclusion

在后端的这些部分，Vulkan的低级显式特性表现出色。我认为我们必须相当低级，否则我们就会继承大多数旧API的问题。

### … up next!

在接下来的部分，我们将看一下管线的创建。