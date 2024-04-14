# Render graphs and Vulkan — a deep dive

现代图形 API，如 Vulkan 和 D3D12，给引擎开发者带来了新的挑战。虽然这些 API 大大减少了 CPU 开销，但很明显，在命中驱动程序的“好”路径并且 GPU 受限时，要在 GPU 性能上跨越差距是很困难的。OpenGL 和 D3D11 驱动程序极尽所能地改善 GPU 性能，使用各种技巧。作为开发者，我们为此付出的代价是无法预测的性能和更高的 CPU 开销。编写图形后端变得更加有趣，因为我们仍在探索如何为这些 API 构建平衡灵活性、性能和易用性的出色渲染后端。

上周，我发布了我的副业项目 Granite，这是我对 Vulkan 渲染引擎的看法。虽然野外有许多类似的项目，各有优点，但我想特别讨论我的渲染图实现。

渲染图的实现受到 Yuriy O’Donnell 在 GDC 2017 的演讲的启发：“FrameGraph: Frostbite 中的可扩展渲染架构。”虽然这次演讲集中在 D3D12 上，但我为 Vulkan 实现了自己的版本。

（注意：这里的渲染图和帧图意思相同。另外，如果我提到 Vulkan，可能也适用于 D3D12 … 或许）

## The problem

Render graphs从根本上解决了现代 API 中一个非常让人讨厌的问题。我们如何处理手动同步？让我们来看看显而易见的替代方案。

### Just-in-time synchronization

最直接的方法基本上是在最后一刻进行同步。每当我们开始向纹理渲染、绑定资源或类似操作时，我们需要问自己，“这个资源是否有待同步的工作？” 如果是，我们需要在最后一刻处理它。这种跟踪显然变得非常痛苦，因为我们可能读取一个资源1000多次，而只写入一次。多线程变得非常痛苦，如果两个线程发现需要障碍？一个线程需要“获胜”，现在我们有很多无用的跨线程同步麻烦要处理。

我们还需要跟踪的不仅仅是执行本身，还有 Vulkan 中的图像布局和内存访问问题。以特定方式使用资源将需要特定的图像布局（或者只是 GENERAL，但你可能会丢失帧缓存压缩！）。

基本上，如果我们想要的是即时自动同步，我们基本上又想要 OpenGL/D3D11。驱动程序已经为此进行了无数次优化，那么为什么我们要以一种半吊子的方式重新实现它呢？

### Fully explicit synchronization

在另一端的 API 抽象中，我们选择完全移除了自动同步，应用程序需要手动处理每个同步点。如果出现错误，准备进行一些“有趣”的调试会话。

对于简单的应用程序来说，这样做是可以的，但一旦你开始这样做，你很快就会意识到它会变得一团糟。通常，你的渲染流水线会被划分为各种模块 — 也许你有一个模块用于正向/延迟/当下流行的任何渲染器，其他模块中散布着一些后处理步骤，也许你会拖入一些反馈用于重新投影步骤，你添加了一种新的技术，你意识到你必须重新制定你的同步策略 — 再一次，事情变得糟糕起来。

为什么会这样？

让我们为一个死简单的后处理步骤编写一些伪代码，然后思考一下。

```
// When was the last time I read from this image? Probably last frame later in the post-chain ...
// We want to avoid write-after-read hazards.
// We're going to write the whole image,
// so we might as well transition from UNDEFINED to "discard" the previous content ...
// Ideally I would keep careful track of VkEvents from earlier frames, but that got so messy ...
// Where was this render target allocated from?
BeginRenderPass(RT = BloomThresholdBuffer)

// This image was probably written to in the previous pass, but who knows anymore.
BindTexture(HDR)

DrawMyQuad()
EndRenderPass()
```

这些问题通常通过一个大而明显的管线屏障来解决。管线屏障让您可以在局部上推理全局同步问题，但它们并不总是最优的解决方式。

```
// To be safe, wait for all fragment execution to complete, this takes care of write-after-read and syncing the HDR render pass ...
// Assuming they are never used in async compute ... hm, this will probably work fine for now.

PipelineBarrier(FRAGMENT -> FRAGMENT,
    RT layout: UNDEFINED -> COLOR_ATTACHMENT_OPTIMAL,
    RT srcAccess: 0 (write-after-read)
    RT dstAccess: COLOR_ATTACHMENT_WRITE_BIT,
    HDR layout: COLOR_ATTACHMENT_OPTIMAL -> SHADER_READ_ONLY,
    HDR srcAccess: COLOR_ATTACHMENT_WRITE_BIT,
    HDR dstAccess: SHADER_READ_BIT)

BeginRenderPass(...)
```

所以我们转换了 HDR 图像，因为我们假设它是先前的通道，但也许在未来你会在其中添加另一个不同的通道，它也会进行转换... 所以现在你仍然需要跟踪图像布局，啊，但也不是世界末日。

如果你只处理片段到片段的工作负载，那么这可能还不算太糟，毕竟在渲染通道之间并没有太多重叠的情况。当你开始将计算引入其中时，你就会开始抓狂，因为你不能随便在各处放置管线屏障，你需要一些关于你的帧的非局部知识，以实现最佳的执行重叠。此外，你甚至可能需要信号量，因为现在你正在不同队列中执行异步计算。

## Render graph implementation

我将主要参考这些文件：render_graph.hpp 和 render_graph.cpp。

注意：这是一个大脑倾泻。在阅读时尽量跟上代码，我会按顺序讲解事情。

注意 #2：在实现中，我使用了“flush”和“invalidate”这两个术语。这不是 Vulkan 规范的术语。Vulkan 分别使用术语“make available”和“make visible”。Flush 是指缓存刷新，invalidate 是指缓存失效。

基本思想是我们有一个“全局”渲染图。系统中所有需要渲染东西的组件都需要向这个渲染图注册。我们指定了我们有哪些通道，哪些资源进入，哪些资源被写入等等。这可以在应用程序启动时执行一次，每帧执行一次，或者根据需要执行。主要思想是我们对整个帧形成全局的了解，我们可以在更高的层次上进行相应的优化。模块可以在本地推理它们的输入和输出，同时允许我们看到更大的图景，这解决了当后端 API 不自动调度并处理我们的依赖关系时我们面临的主要问题。渲染图可以处理屏障、布局转换、信号量、调度等等。

渲染通道的输出需要一些维度，相当简单。

Images:

```
struct AttachmentInfo
{
	SizeClass size_class = SizeClass::SwapchainRelative;
	float size_x = 1.0f;
	float size_y = 1.0f;
	VkFormat format = VK_FORMAT_UNDEFINED;
	std::string size_relative_name;
	unsigned samples = 1;
	unsigned levels = 1;
	unsigned layers = 1;
	bool persistent = true;
};
```

Buffers:

```
struct BufferInfo
{
	VkDeviceSize size = 0;
	VkBufferUsageFlags usage = 0;
	bool persistent = true;
};
```

These resources are then added to render passes.

```
// A deferred renderer setup

AttachmentInfo emissive, albedo, normal, pbr, depth; // Default is swapchain sized.
emissive.format = VK_FORMAT_B10G11R11_UFLOAT_PACK32;
albedo.format = VK_FORMAT_R8G8B8A8_SRGB;
normal.format = VK_FORMAT_A2B10G10R10_UNORM_PACK32;
pbr.format = VK_FORMAT_R8G8_UNORM;
depth.format = device.get_default_depth_stencil_format();

auto &gbuffer = graph.add_pass("gbuffer", VK_PIPELINE_STAGE_ALL_GRAPHICS_BIT);
gbuffer.add_color_output("emissive", emissive);
gbuffer.add_color_output("albedo", albedo);
gbuffer.add_color_output("normal", normal);
gbuffer.add_color_output("pbr", pbr);
gbuffer.set_depth_stencil_output("depth", depth);

auto &lighting = graph.add_pass("lighting", VK_PIPELINE_STAGE_ALL_GRAPHICS_BIT);
lighting.add_color_output("HDR", emissive, "emissive");
lighting.add_attachment_input("albedo");
lighting.add_attachment_input("normal");
lighting.add_attachment_input("pbr"));
lighting.add_attachment_input("depth");
lighting.set_depth_stencil_input("depth");

lighting.add_texture_input("shadow-main"); // Some external dependencies
lighting.add_texture_input("shadow-near");
```

这里我们看到资源在渲染通道中可以被使用的三种方式。

1. 只写：资源完全被写入。对于渲染目标，loadOp = CLEAR 或 DONT_CARE。
2. 读写：保留一些输入，并在其上写入，对于渲染目标，loadOp = LOAD。
3. 只读：显而易见。

计算通道的情况类似，这是一个自适应亮度更新通道，在异步计算中完成。

```
auto &adapt_pass = graph.add_pass("adapt-luminance", VK_PIPELINE_STAGE_COMPUTE_SHADER_BIT);
adapt_pass.add_storage_output("average-luminance-updated", buffer_info, "average-luminance");
adapt_pass.add_texture_input("bloom-downsample-3");
```

在这里，例如，亮度缓冲区进行了 RMW。

我们还需要一些回调函数，可以在每帧调用，来实际做一些工作，比如对 GBuffer...

```
gbuffer.set_build_render_pass([this, type](Vulkan::CommandBuffer &cmd) {
	render_main_pass(cmd, cam.get_projection(), cam.get_view());
});

gbuffer.set_get_clear_depth_stencil([](VkClearDepthStencilValue *value) -> bool {
	if (value)
	{
		value->depth = 1.0f;
		value->stencil = 0;
	}
	return true; // CLEAR or DONT_CARE?
});

gbuffer.set_get_clear_color([](unsigned render_target_index, VkClearColorValue *value) -> bool {
	if (value)
	{
		value->float32[0] = 0.0f;
		value->float32[1] = 0.0f;
		value->float32[2] = 0.0f;
		value->float32[3] = 0.0f;
	}
	return true; // CLEAR or DONT_CARE?
});
```

渲染图负责分配资源并驱动这些回调函数，最终按正确顺序将其提交到 GPU。为了终止这个图，我们将特定的资源提升为“后备缓冲区”。

```
// This is pretty handy for ad-hoc debugging :P
const char *backbuffer_source = getenv("GRANITE_SURFACE");
graph.set_backbuffer_source(backbuffer_source ? backbuffer_source : "tonemapped");
```

Now let’s get into the actual implementation.

## Time to bake!

一旦我们设置了结构，我们需要烘焙渲染图。这经历了一系列步骤，每个步骤完成一个谜题的一部分...

### Validate

相当直接，进行快速的健全性检查，以确保 RenderPass 结构中的数据是合理的。

这里有一件有趣的事情，我们可以检查颜色输入的尺寸是否与颜色输出匹配。如果它们不同，我们就不会直接使用 loadOp = LOAD，而是在渲染通道开始时进行比例缩放的 blit。对于像游戏渲染低分辨率 -> 原生分辨率 UI 这样的情况，这非常方便。在这种情况下，loadOp 变为 DONT_CARE。

### Traverse dependency graph

现在我们有了一个渲染通道的无环图（希望如此… :D），我们需要将其展平为一个渲染通道数组。我们创建的列表将是一个有效的提交顺序，如果我们一个接一个地提交每个通道的话。这个提交顺序可能不是最优的，但我们稍后会接近最优。

这里的算法很简单。我们从底向上遍历树。使用递归，将写入后备缓冲区的所有通道的通道索引推入列表，然后对于所有这些通道，将这些通道中的资源写入推入列表... 依此类推，直到到达顶层叶子节点。通过这种方式，我们确保如果通道 A 依赖于通道 B，那么通道 B 将始终在列表中出现在通道 A 之后。现在，反转列表，并修剪重复项。

我们还注册了一个通道是否是另一个通道的良好“合并候选者”。例如，光照通道使用 gbuffer 通道的输入附件，并共享一些颜色/深度附件... 在基于瓦片的体系结构上，我们实际上可以使用 Vulkan 的多通道功能合并这些通道，而无需访问主内存，因此我们会在稍后的重新排序通道中考虑这一点。

### Render pass reordering

这是整个过程中的第一个有趣步骤。理想情况下，我们希望有一个提交顺序，使通道之间的重叠最大化。如果通道 A 写入一些数据，而通道 B 读取它，我们希望在 A 和 B 之间有尽可能多的通道，以减少“硬屏障”的数量。这成为我们的优化指标。

实现的算法可能在 CPU 时间方面非常不优化，但它完成了任务。它遍历尚未安排的通道列表，并根据三个标准尝试找出最佳通道：

1. 根据先前的依赖图遍历步骤确定我们是否有合并候选者？（分数：无穷大）
2. 已安排的通道列表中，我们需要等待的最新通道是什么？（分数：在两者之间可以重叠的通道数量）
3. 安排此通道是否会破坏依赖链？（如果是，则跳过此通道）。

阅读代码可能更具教育意义，请参阅 RenderGraph::reorder_passes()。

另一个需要考虑的巧妙问题是，当光照通道依赖于某些资源时，而 G-buffer 通道则没有。这可能会破坏子通道合并，因为我们经历了这个调度过程：

1. 安排 G-buffer 通道，它没有依赖关系。
2. 尝试安排光照通道，但糟糕，我们还没有安排我们依赖的阴影通道 ... 哎呀 🙂

这个问题的简单解决方案是，将合并候选者的依赖关系提升到合并链中的第一个通道。因此，G-buffer 通道将在阴影通道之后安排，一切都很好。一个更聪明的调度算法可能会在这里有所帮助，但我想尽可能保持简单。

### Logical-to-physical resource assignment

当我们构建我们的图时，我们可能会有一些读取-修改-写操作。对于光照通道，发射光进入，HDR 结果输出，但显然，它实际上是同一个资源，我们只是有这个抽象来以合理的方式确定依赖关系，给资源命名一些描述性名称，并避免循环。如果我们有多个通道，都执行发射光 -> 发射光，例如，我们不知道哪个通道先执行，它们彼此都依赖（？），我宁愿不处理潜在的循环。

现在，我们为所有资源分配一个物理资源索引，并为执行读取-修改-写操作的资源进行别名处理。如果由于某种原因我们无法进行别名处理，这表明我们有一个非常奇怪的提交顺序，试图在写入同时进行读取。在这种情况下，实现会放弃处理。我不认为这会在一个无环图中发生，但我无法证明。

### Logical-to-physical render pass assignment

接下来，我们尝试将相邻的渲染通道合并在一起。这在基于瓦片的渲染器上特别重要。如果满足以下条件，我们尝试将通道合并在一起：

1. 它们都是图形通道。
2. 它们共享一些颜色/深度/输入附件。
3. 不存在多个唯一的深度/模板附件。
4. 它们的依赖关系可以使用 BY_REGION_BIT 实现，即没有“纹理”依赖，这允许对任意位置进行采样。

### Transient or physical image storage

类似于子通道合并，基于瓦片的渲染器可以避免为附件分配物理内存，如果您实际上从不写入它（即 storeOp = STORE）的话！这对延迟渲染尤其有很大的内存节省，但对于深度缓冲区，如果它们在后期没有被使用，也是一样的。

资源可以是瞬态的，如果满足以下条件：

1. 它在单个物理渲染通道中使用（即它从不需要 storeOp = STORE）。
2. 它在渲染通道开始时被无效化（不需要 loadOp = LOAD）。

### Build RenderPassInfo structures

现在，我们清楚地了解了所有通道，它们的依赖关系等等。现在是创建一些渲染通道信息结构的时候了。

这部分实现与 Granite 的 Vulkan 后端密切相关，但它紧密地反映了 Vulkan API，所以不应该太奇怪。在 Vulkan 后端中，VkRenderPasses 是按需生成的，所以我们在这里不这样做，但我们现在可以潜在地完成这个过程。

实际的图像视图将稍后分配（实际上是每帧），但是子通道信息、颜色附件的数量、输入、用于 MSAA 的解析附件等可以提前完成。我们还构建了一个列表，指示应该作为附件拉入的哪些物理资源索引。

我们还通过调用一些回调函数来确定哪些附件现在需要 loadOp = CLEAR 或 DONT_CARE。对于具有输入的附件，只需使用 loadOp = LOAD（或使用比例缩放的 blit！）。对于 storeOp，我们总是使用 STORE。Granite 内部识别瞬态附件，并且无论如何都会强制 storeOp = DONT_CARE。

### Build barriers

现在是开始查看屏障的时候了。对于每个通道，每个资源经历三个阶段：

1. 转换到适当的布局，需要使缓存失效。
2. 资源被使用（读取和/或写入发生）。
3. 资源以新布局结束，可能需要稍后刷新的写入。

对于每个通道，我们构建一个“使失效”和“刷新”的列表。

通道的输入被放入失效桶中，输出被放入刷新桶中。读取-修改-写入资源将在两个桶中都有条目。

例如，如果我们想在通道中读取纹理，我们可能会添加这样的失效屏障：

- stages = FRAGMENT（或者好吧，VERTEX，但我必须为资源输入添加额外的阶段标志）
- access = SHADER_READ
- layout = SHADER_READ_ONLY_OPTIMAL

对于颜色输出，我们可能会这样说：

- stages = COLOR_ATTACHMENT_OUTPUT
- access = COLOR_ATTACHMENT_WRITE
- layout = COLOR_ATTACHMENT_OPTIMAL

这告诉系统：“嘿，这个阶段有一些待写入的数据，使用这个内存访问需要刷新与 srcAccessMask 相关的内容。如果你想使用这个资源，与这些东西同步！”

我们还可以在这里确定一个特定的场景与渲染通道。如果一个资源既用作输入附件又用作只读深度附件，我们可以将布局设置为 DEPTH_STENCIL_READ_ONLY_OPTIMAL。如果颜色附件也用作输入附件，我们可以使用 GENERAL（可编程混合！），类似的用于读写深度/模板与输入附件。

### Build physical render pass barriers

现在，我们对每个通道的屏障有了完整的了解，但是当我们开始合并通道时会发生什么呢？多通道可能会在渲染通道执行过程中执行一些内部屏障（考虑延迟着色），因此我们可以在这里省略一些屏障。这些屏障将在稍后构建 VkRenderPass 时通过 VkSubpassDependency 在内部解决，因此我们可以忘记所有需要在子通道之间发生的屏障。

我们关心的是为资源第一次使用构建失效屏障。对于刷新屏障，我们关心资源的最后一次使用。

现在，这里有两种情况我们需要处理，以确保每个通道都能在通道执行之前和之后处理同步。

#### Only invalidation barrier, no flush barrier

这是只读资源的情况。我们仍然需要防范后续写入-读取危险。例如，如果下一个通道开始向此资源写入，我们显然需要让其他通道知道，必须在我们开始在资源上涂鸦之前完成此通道。这种实现方式是通过插入一个虚假的刷新屏障，access = 0。access = 0 基本上意味着：“这里没有待处理的写入！”这样，我们可以连续进行多个读取资源的通道。如果图像布局保持不变且 srcAccessMask 为 0，则我们不需要屏障。

#### Only flush barrier, no invalidation barrier

这通常是“只写”通道的情况。这让我们知道，在通道开始之前，我们可以通过从 UNDEFINED 进行转换来丢弃资源。然而，我们仍然需要一个失效屏障，因为在开始渲染通道之前，我们需要进行布局转换，缓存需要失效，因此我们在此处插入一个具有相同布局和访问权限的失效屏障。

#### Ignore barriers for transients/swapchain

你可能注意到，某些原因造成了瞬态资源的屏障被“丢弃”。Granite 在内部使用外部子通道依赖关系来执行瞬态附件的布局转换，尽管这现在可能有些多余，因为渲染图。交换链类似。Granite 在内部使用子通道依赖关系来在渲染通道中使用交换链图像时将其转换为 finalLayout = PRESENT_SRC_KHR

### Render target aliasing

在我们的烘焙过程的最后一步是确定我们是否可以在图中对资源进行时间上的别名。例如，我们可能有两个或更多资源，在帧中完全不同的时间存在。考虑一个可分离的模糊：

1. 渲染一帧（缓冲区＃0）
2. 水平模糊（缓冲区＃1）
3. 垂直模糊（应该返回到缓冲区＃0）
当我们在渲染图中指定此过程时，我们有3个不同的资源，但是显然，垂直模糊渲染目标可以与初始渲染目标别名。我建议查看Frostbite关于别名的演示，它的效果非常显著。

在这里，我们可以在实际的 VkDeviceMemory 中进行别名，但是此实现只是尝试直接重用 VkImages 和 VkImageViews。我不确定直接从其他图像的死尸中进行子分配并希望它能够奏效是否会有太大好处。我猜想，如果你真的对内存非常渴望，可以考虑这种别名图像内存的做法。别名图像内存的优点可能是可疑的，因为 VK_*_dedicated_allocation 是一个选择，因此某些实现可能更喜欢您不进行别名。这显然需要一些数字和 IHV 指导。

算法相当简单。对于每个资源，我们确定资源被使用的第一个和最后一个物理渲染通道。如果我们找到另一个具有相同尺寸/格式的资源，并且它们的通道范围不重叠，那么我们可以进行别名！我们注入一些信息，可以在资源之间进行“所有权”转移。

例如，如果我们有三个资源：

- 别名＃0 在通道＃1和＃2中使用
- 别名＃1 在通道＃5和＃7中使用
- 别名＃2 在通道＃8和＃11中使用

在通道＃2结束时，与别名＃0相关的屏障被复制到别名＃1，并且布局被强制为 UNDEFINED。当我们开始通道＃5时，我们会在将图像转换为新布局之前神奇地等待通道＃2完成。别名＃1在通道＃7后将控制权交给别名＃2，依此类推。通道＃11以“环”的方式将控制权交还给别名＃0的下一个帧。

这里有一些注意事项。某些图像可能具有“历史”或“反馈”，其中每个图像实际上有两个实例，一个用于当前帧，另一个用于前一帧。这些图像不应与其他任何内容别名。此外，瞬态图像不进行别名。Granite 的内部瞬态图像分配器会在内部处理此别名，但是现在有了渲染图，这似乎有些多余…

另一个考虑因素是，添加别名可能会增加所需的屏障数量，并减少 GPU 的吞吐量。也许别名代码需要考虑额外的屏障成本？唉…至少如果您在烘焙时知道了您的 VRAM 大小，您可以根据图中所有资源来确定别名是否真正值得。将依赖图优化为最大重叠也极大地减少了别名的机会，因此，如果我们想考虑内存，这种算法可能会变得更加复杂…

### Preparing resources for async compute

对于异步计算，资源可能会被图形和计算队列访问。如果它们的队列族不同（嗨，AMD），我们必须决定是否希望这些资源在队列之间具有独占或并发访问。对于缓冲区，使用并发访问似乎是一个明显的选择，但对于图像来说就有点复杂了。为了不使这变得非常复杂，我选择了并发访问，但仅适用于真正需要在计算和图形通道中使用的资源。处理独占访问将是很困难的，因为现在我们还必须考虑读取后读取的屏障，并在两个队列族之间进行所有权的来回交换（哦亲爱的）。

### Summary

考虑到的内容确实很多，但现在我们已经准备好开始生成帧了，所有的数据结构都已经就绪。

## The runtime

虽然烘焙过程非常复杂，但执行起来相对简单，我们只需要跟踪图中所有已知资源的状态。

每个资源都存储着：

1. 最后一个 VkEvent。如果我们需要问自己：“在我触及这个资源之前我需要等待什么”，那么这就是答案。我选择了 VkEvent，因为它可以表达执行重叠，而管线屏障则不能。
2. 图形队列的最后一个 VkSemaphore。如果资源在异步计算中使用，我们使用信号量而不是 VkEvent。信号量不能多次等待，因此如果需要的话，我们在图形队列中使用一个信号量进行一次等待。
3. 计算队列的最后一个 VkSemaphore。相同的情况，但是在计算队列中等待一次。
4. 刷新阶段（VkPipelineStageFlags），包含了我们需要等待的阶段（srcStageMask），如果需要等待资源。
5. 刷新访问（VkAccessFlags），包含了我们需要在使用资源之前刷新的内存的 srcAccessMask。
6. 每个阶段的失效标志（对于每个管线阶段的 VkAccessFlag）。这些位掩码跟踪着在哪些管线阶段和访问标志下可以安全使用资源。如果我们确定需要失效屏障，但是所有相关的阶段和访问位都已经准备就绪，那么我们可以完全丢弃该屏障。这对于我们反复读取相同资源，且处于 SHADER_READ_ONLY_OPTIMAL 布局的情况非常有用。
7. 资源的当前布局。目前，布局存储在图像句柄内部，但如果以后添加了多线程，这可能会有些困难。
8. 对于每一帧，我们分配资源。至少我们必须替换交换链图像，但是某些图像可能被指定为“非持久”状态，在这种情况下，我们每一帧都分配一个新资源。这对于我们希望消除所有跨帧屏障的场景非常有用，即以更多的内存使用换取更多的 GPU 内存中的拷贝。对于大型渲染目标来说，这可能是一个糟糕的主意，但对于每个几 KB 的小计算缓冲区来说呢？当然。如果我们能尽早启动 GPU 工作，那可能是件好事。

如果我们分配了一个新的资源，则所有的屏障状态都会被清除为初始状态。

现在，我们开始推出渲染通道。当前的实现遍历所有通道，并在屏障出现时进行处理。如果你非常巧妙地交错这个循环，我相信你会看到一些多线程的潜力。

### Check conditional execution

某些渲染通道不需要在此帧运行，并且可能只需要在发生某些事件时运行（比如阴影地图）。每个通道都有一个回调函数来确定这一点。如果一个通道没有被执行，它就不需要失效/刷新屏障。我们仍然需要处理别名屏障，所以只需这样做，然后转到下一个通道即可。

### Handle discard barriers

如果一个通道具有丢弃屏障，只需将图像的当前布局设置为 UNDEFINED。当我们实际执行布局转换时，我们将使用 oldLayout = UNDEFINED。

### Handle invalidate barriers

这一部分的关键是确定我们是否需要使某些缓存失效，并可能刷新某些缓存。在这里，我们需要检查一些事项：

- 是否有待处理的刷新？
- invalidate 屏障是否需要与当前图像布局不同的图像布局？
- 是否有一些缓存尚未被刷新？

如果对任何一个问题的答案是肯定的，我们就需要某种形式的屏障。我们以以下三种方式实现这种屏障：

- vkCmdWaitEvents – 如果资源有待处理的 VkEvent，还需要适当的 VkBufferMemoryBarrier/VkImageMemoryBarrier。
- vkQueueSubmit w/ 信号等待。Granite 在提交时负责添加信号量。我们推入一个等待信号量，以及与我们的 invalidate 屏障相匹配的 dstWaitStageMask。如果我们还需要布局转换，我们可以添加一个 vkCmdPipelineBarrier，其 srcStageMask = dstStageMask 以锁定到 dstWaitStageMask … 并保持管线的运行。如果我们在信号等待时不需要处理 srcAccessMask，通常这将被强制为 0。
- vkCmdPipelineBarrier(srcStage = TOP_OF_PIPE_BIT)。如果资源以前未被使用，并且我们只需从 UNDEFINED 布局转换即可使用。

屏障将适当地批处理并提交。缓冲区要简单得多，因为它们没有布局。

在失效后，我们将适当的阶段标记为正确失效。如果我们在此步骤中更改了布局或刷新了内存访问，则在此步骤之前将所有内容清除为 0。

### Execute render passes

这部分比较简单，只需调用 begin/nextsubpass/end 并触发一些回调来执行真正的图形工作。对于计算，只需忽略 begin/end。

对于图形工作，在开始时可能会进行一些缩放 blit，而在结束时可能会进行一些自动 mipmap 生成。

### Handle flush barriers

这部分比较简单。如果至少有一个资源仅在单个队列中使用，我们会在这里发出一个 VkEvent，并将其分配给所有相关资源。如果我们至少有一个资源在跨队列中使用，我们也会在这里发出两个信号量（一个用于图形，另一个用于稍后的计算…）

我们还会更新当前布局，并标记刷新阶段/刷新访问以供稍后使用。

### Alias handoff

如果资源被别名化，我们现在将资源的屏障状态复制到其下一个别名，并强制布局为 UNDEFINED。

### Submission

每个通道的命令缓冲区现在被提交到 Granite。Granite 尝试将命令缓冲区批处理，直到需要等待信号量或发送信号量为止。

### Scale to swapchain

在所有通道完成后，如果后备缓冲区资源的尺寸与实际交换链不匹配，我们可以向交换链注入最后的 blit。否则，无论如何我们都会将这些资源别名化，因此不需要无用的 blitting 通道。

## Conclusion

这个讨论确实很有趣。到目前为止，这篇文章的字数接近 5,000 字，而渲染图则是一个 3,000 行代码的庞然大物（叹息）。我相信还会有一些 bug（事实上，我在写这篇文章的时候就发现了两个异步计算的 bug），但我对结果感到非常满意。

未来的目标可能是尝试将其制作成可重用的独立库，并获取一些实际的数据。