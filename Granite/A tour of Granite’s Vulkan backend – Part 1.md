# A tour of Granite’s Vulkan backend – Part 1

## Introduction


自2017年1月以来，我一直在进行我的小引擎项目，我称之为Granite。它在Github上。像许多人一样，我觉得我需要为自己编写一个小引擎来完全学习Vulkan，并且我需要一个测试平台来实现各种图形技术。从那时起，我一直在稳步地进行开发，并将其用作许多副项目的基础，但我认为对于其他人来说，它目前的价值在于通过示例来教授Vulkan的概念。

不久前，我写了一篇关于我的渲染图实现的博客。渲染图位于Vulkan实现的顶部，但在这个系列中，我想专注于Vulkan层本身。

### The motivation for a useful mid-level abstraction

在过去几年的Twitter圈和各种专题讨论中，我注意到了一个中层抽象的概念。在非平凡应用中，GL和D3D11的水平过高且不灵活，而Vulkan和D3D12往往在低级复杂性方面过度发展，其目标是尽可能低级和明确，同时保持GPU架构/操作系统的可移植性。我认为每个人都同意拥有一个良好的中层抽象很重要，但我们在设计这些层时总是面临着如何做出正确权衡的问题。总会有一些人追求最大可能的性能，即使在使用抽象时复杂性飙升。

对于Granite，我总是希望提高便利性，同时避免在性能方面遭受最严重的惩罚。基本上是那个老掉牙的80/20法则。在Vulkan中有许多许多机会可以不做冗余的CPU工作，但代价是什么？值得将自己架构成一个钻石吗？一个超级坚固但最终不灵活的实现？我注意到人们普遍在这个主题周围感到很焦虑，特别是初学者。对于不追求每一个可能的性能优化的恐惧是普遍存在的，因为“以后可能会非常重要”，这可能是我们还没有看到广泛使用的标准中层图形API的原因。

我觉得通过为最大可能的CPU性能设计而获得的好处更多是理论设计练习而不是实践。根据我的经验，即使是天真的、直截了当的、单线程的Vulkan在CPU开销方面也远远超过GL/GLES，仅仅因为我们可以挑选并选择我们需要做的额外工作，但是传统的驱动堆栈已经在十年或更长时间内积累了大量的垃圾来支持各种奇怪的用例和启发式方法。在此基础上增加多线程，然后您可以考虑微调API开销，如果您确实需要的话。我怀疑您甚至不需要多线程Vulkan。我相信当前现代API的真正挑战是优化GPU性能，而不是CPU。

Metal因其在性能和可用性方面的成功权衡而受到了很多赞扬。虽然我不详细了解API本身以做出判断（尽管我知道Metal Shading Language的所有恐怖之处），但我印象中中层抽象是我们应该99%的时间内工作的抽象层级。

我认为Granite是这样的一个实现。我并不试图提出Granite是解决方案，但它是其中之一。设计空间是巨大的。不可能有一个真正适合所有用户的图形API。我不建议直接使用它，而是将尝试解释我是如何设计一个在桌面和移动设备上运行良好（很少有项目考虑到这两者）的Vulkan接口，至少对于我的用例来说。理想情况下，您应该受到启发，制定适合您和您的项目的中层抽象。我经历了几次迭代才达到目前的设计，并将其用于各种项目，所以我认为至少是一个很好的起点。

#### The 3D-accelerated emulation use case


Granite的起步实际上是Beetle PSX HW渲染器中的Vulkan后端。我编写了一个Vulkan后端，模拟器需要非常及时和灵活地使用图形API。信息通常只在最后一刻才知道。能够实现这样的项目极大地指导了Granite的初始设计过程。这也是一个情况，传统API确实非常痛苦，因为您确实需要现代API的灵活性才能在性能方面做得很好。在模拟器本身的CPU成本之上，状态更改和绘制调用非常频繁。在模拟中以奇怪的方式创建资源和修改GPU上的数据是常见情况，许多驱动程序根本无法理解这些使用模式，我们到处都会遇到痛苦的慢路径。使用Vulkan几乎没有什么魔法，我们只是按照自己的想法实现事情，性能最终变得更加可预测。

我认为很多人忘记了Vulkan不仅仅适用于大型（AAA）游戏引擎。我们可以成功地将其用于各种事情。我们只需要正确的抽象和知识。

### How the design and implementation will be explored

To start off, we will explore the design through commented code samples, which use only the Vulkan portion of Granite as a library. We will write concrete samples of code, and then go through how all of this works, and then discuss how things could be designed differently.