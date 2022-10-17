---
title: subpass dependency 摘录
date: 2022年9月22日
tags: vulkan
---

原文链接：
https://www.reddit.com/r/vulkan/comments/s80reu/subpass_dependencies_what_are_those_and_why_do_i/?sort=top

<!-- more -->

下面是翻译：

Subpass Dependencies：那些是什么，为什么我需要它们？

你好，

简而言之，我研究了子通道依赖关系，我越深入这个主题就越复杂。我的理解是，这种依赖关系对在渲染帧时同步信号量有很大帮助。比如说，我们启动一个帧并尝试从交换链获取图像。我们等待信号量并继续。之后，我们记录命令缓冲区，最后将其提交到队列并呈现。

这些步骤本质上确实需要一个信号量，我的猜测是，这些子通道依赖项有助于 gpu 将我们的命令同步到 gpu。

如果是这样的话......为什么它在“subpasses”之后命名。它必须与子通道有关。

我对 vulkan api 非常陌生。我之前弄乱了opengl并用它创建了一个游戏引擎。但是这个 api 让我很困惑......我也知道桌面 gpu 几乎不需要子通道。如果我的目标平台本质上是桌面，我应该使用子通道吗？

任何人都可以指导我正确的方向吗？

谢谢



回复翻译：

你不是一个人; 在我看来，这是 Vulkan 最复杂/最困难的部分之一。

基本上，您拥有由**附件**、**子通道**和**子通道依赖项组成的****渲染通道**。

**附件**基本上只是可以在渲染过程中使用的图像，例如作为着色器输入（“输入附件”）或渲染目标（“颜色附件”）。

**子通道**是一组渲染命令，以及在这些渲染命令中使用的所有附件的“引用”列表。

那么，什么是子通道依赖项？好吧，假设您有两个子通道：

1. 将某些内容渲染为附件的子通道。
2. 使用该附件作为着色器输入的子通道，用于渲染其他内容。

在此示例中，对第一个子通道中的第一个附件进行的渲染需要在它用作第二个子通道中的着色器输入*之前完成。*否则渲染结果会一团糟！

问题是，默认情况下，子通道实际上不会以任何特定顺序运行。出于性能原因，它们可以按任何顺序运行。在某些情况下，这完全没问题。但是，在我们的例子中，我们实际上*需要*这些子通道按照特定的顺序进行，否则渲染将被搞砸！

这就是子通道依赖关系的来源。子通道依赖关系几乎就是它们听起来的样子：子通道之间的依赖关系。它们只是让我们指定我们需要强制事情发生的顺序，以获得我们想要的结果。

在上面的示例中，我们需要一个子通道依赖项，如下所示：

```
VkSubpassDependency dep;
dep.srcSubpass = 0;
dep.dstSubpass = 1;

dep.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dep.dstStageMask = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;

dep.srcAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
dep.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;
```

**srcSubpass**是我们依赖的子通道的索引。在这种情况下，我们需要在第一个子通道完成执行之前不启动第二个子通道，因此我们将 srcSubpass 设置为 0。

如果我们想依赖作为先前渲染通道一部分的子通道，我们可以在这里传入 VK_SUBPASS_EXTERNAL。在这种情况下，这意味着“在此之前等待所有渲染通道中的所有子通道”。

**dstSubpass**是当前子通道的索引，即存在此依赖关系的子通道。在本例中，这是第二个子通道，因此 dstSubpass 为 1。

现在这就是棘手的地方。

**srcStageMask**是所有 Vulkan “阶段”（基本上是渲染过程的步骤）的位掩码，我们要求 Vulkan 在进入 dstSubpass*之前*在 srcSubpass 中完成执行。

在这种情况下，我们希望在第二个子通道中使用通过第一个子通道渲染的颜色输出。因此，我们需要等待第一个子通道完成渲染其颜色输出。或者，换句话说，我们需要等到第一个子通道完成其颜色附件输出阶段。所以，srcSubpass 是 VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT。

**dstStageMask**是 dstSubpass 中所有 Vulkan 阶段的位掩码，直到srcStageMask 中的阶段在 srcSubpass 中完成*后*我们才能执行。

通过传入 VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT，我们告诉 Vulkan 在 dstSubpass 中可以按照它想要的任何顺序执行任何命令，除了与片段着色器相关的命令。在 srcStageMask 阶段在 srcSubpass 中完成执行之前，我们故意不允许 Vulkan 执行这些命令。

好的，现在我们已经指定了两个子通道之间的*执行依赖关系*（这些子通道必须执行的顺序）。但是 GPU 是复杂的野兽，它会执行大量的图像缓存等操作，因此仅指定我们需要这些渲染命令实际发生的顺序是不够的。我们还需要告诉 Vulkan 我们需要的内存访问类型以及何时需要它们，以便它可以相应地更新缓存等。

因此，**srcAccessMask**是 srcSubpass 使用的所有 Vulkan 内存访问类型的位掩码，而**dstAccessMask**是我们将在 dstSubpass 中使用的所有 Vulkan 内存访问类型的位掩码。

想象一下，就像我们在说：“在您完成对 srcSubpass*中颜色附件*的写入后，根据需要‘刷新’结果，以便我们能够*在我们的着色器中读取它*。”

所有这些值都可以根据需要被弄乱，以使 Vulkan 基本上同步子通道/渲染通道，但是你想要它。这一切都非常复杂，但也非常强大。就像大多数 Vulkan 一样。

希望这一切都有助于更好地解释事情！

如果您想了解更多信息，除了人们已经链接的其他页面之外，我建议您观看[关于渲染通道的精彩演讲](https://youtu.be/x2SGVjlVGhE)并阅读[Vulkan 规范中的第 8 章](https://www.khronos.org/registry/vulkan/specs/1.2-khr-extensions/html/chap8.html)。
