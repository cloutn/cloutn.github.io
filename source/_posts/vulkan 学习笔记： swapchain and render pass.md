---
title: vulkan 学习笔记： swapchain and render pass
date: 2022年9月19日
tags: vulkan
---

## 一、概述

这篇文章的起因是，想在引擎中添加点击点选物体的操作，在不接入物理引擎的前提下，就需要使用将场景物件着色渲染到一个 render target 上，然后从上面取色的方式来确定选中了哪个 3D 物体。这就要求我们要让渲染器支持将场景渲染到一个离线 render target 的功能。

之前的渲染都是直接到屏幕的，所以要进行一系列的调整，使得渲染器支持更丰富的自定义render target。这就要求我们理清 vulkan 中 render pass 和 frame buffer 以及相关的一些概念。之前的 render pass 和主屏幕使用的 frame buffer 都是直接绑定到 svkSwapchain 中的，这仅仅是为了方便隐藏细节，现在因为以上需求必须要进行改造了。

<!-- more -->

关于 3D Picking，知乎上有个答案总结的比较全面：https://www.zhihu.com/question/374183614。
具体的 vulkan 实现有个 rust 的作为参考：https://snoozetime.github.io/2019/05/02/object-picking.html
用 vulkan 实现离屏渲染的代码有例子代码：https://github.com/SaschaWillems/Vulkan/blob/master/examples/offscreen/offscreen.cpp

另外，说一句题外话，未来是一定会接入物理引擎的，目前主流的物理引擎有如下几个：PhysX，Havok，Bullet，Newton Dynamics，不出意外会接入 PhysX。

## 二、render pass

在 vulkan 中，render pass 是多个 subpass 的集合。每个 subpass 需要绑定 attachment，确认这个 pass 最终要渲染到哪里。沿着这个思路，我们看到在 vulkan sdk 自带的 samples 中，创建一个 render pass 的流程如下：

- 创建 image 和 depth 对应的 attachment。注意这里只是描述相关参数，没有涉及到 frame buffer 和实际的 image view 的创建。
- 创建 subpass description，并关联上 attachment
- 描述 subpass dependency。这一步比较难以理解，有几个相关文章可以作为参考：
  - [Vulkan Mobile Best Practices](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/vulkan-best-practices-frequently-asked-questions-part-1)
  - [Vulkan Tutorial](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Render_passes)
- 创建 render pass。

在创建的过程中，我们只需要颜色缓冲区的 format 和深度缓冲区的 format 即可，所以可以看到，这里和 swapchain 以及 framebuffer 没有很强的耦合性。

- 

## 三、swapchain

swapchain 是一个和硬件屏幕相关的一个概念，本质上是为了解决渲染更新到硬件屏幕过程中的撕裂问题。swapchain 要管理多个 render target 的队列，并把这些内容 present 到硬件屏幕上，所以会和 framebuffer 有比较大的关系。
swapchain 必然会管理那些直接渲染到屏幕的 framebuffer，并且因为要双缓冲或者三缓冲，所以往往有2到3套完全一致的 framebuffer，分别用于同一时刻的 present 和 renderer。

基于以上概念，swapchain 的创建会包含很多和硬件相关的概念，比如 surface，就是 OS 对屏幕硬件的一个抽象。再比如swapchain 使用的图像的 format 必须是和 surface 对应的。甚至因为这个概念和硬件更近，离渲染 sdk 本身更远，导致创建 swapchain 的结构体都是一个 KHR 扩展 VkSwapchainCreateInfoKHR。下面就具体列举一下创建一个 swapchain 的步骤，并就这些步骤梳理其和 renderpass 以及 framebuffer 的关系：

- 保证成功创建了 surface，并从 surface 中找到可以 present 的 queue family。
- 获取 surface 的 capability，并根据 capability 和实际需要，确定 swapchain 图像的尺寸，format，color space。
- 填充 VkSwapchainCreateInfoKHR 结构体，创建 swapchain。
- 获取 swapchain 的 image 信息，创建对应的 image view。

可以看到，swapchain 的创建，并不需要实际的 framebuffer，只是需要 image view即可。甚至不需要深度缓冲。因为 swapchain 只是将已经渲染好的画面 present 到屏幕上，所以并不关心也不需要深度信息，甚至不需要一个 framebuffer，只是需要一个更低级的 image 即可。但是这个 image 是从 frame buffer 中渲染出来的，后面我们分析 framebuffer 的时候，会看到这个关联性。总之，从 vulkan 的整体设计来看，swapchain 这部分也是比较独立的，可以脱离 framebuffer 和 renderpass 单独创建。同时我们会在后面看到，创建用于渲染到屏幕的 frame buffer 的时候，我们必须指定 framebuffer 的 image 是来源于 swapchain 的。

## 四、framebuffer

framebuffer 是用于某个 render pass 的多个 image 的**引用的**集合，所以从以上概念中，我们可以得出 framebuffer 的创建过程如下所示：

- 指定 renderpass
- 指定包含了 image 的 attachment，attachment 的**类型是 image view**。
- 指定尺寸。如果是渲染到屏幕，则取自 swapchain 关联的 surface。
- 创建 framebuffer

可以看到 framebuffer 的创建需要 renderpass，同时也需要 swapchain 的 image queue 中的 image 的 image view。以及 swapchain 关联 surface 的尺寸大小。所以 framebuffer 是耦合性比较强的。

但是假如我们渲染到 off screen，有完全不需要swapchain，因为 image 我们可以自己创建一个空纹理，尺寸直接使用空纹理尺寸就可以了，不需要 swapchain，但是我们还是必须要创建一个 renderpass，不过这个 renderpass不是我们的 main render pass，只是用于 off screen 渲染的。

## 五、swapchain 使用的各类信号量

由于 vulkan 的渲染模型是异步的，所以我们在使用一帧图像的时候，必须保证这一帧图形是已经被 present 不需要的，因此我们需要一些同步原语来处理 swapchain 的各种情况。

我们先归纳一下 vulkan 的四种同步原语：

- Fence ：最大粒度同步原语，让 CPU 知道 GPU 什么时候做完工作。
- Semaphore ：比 Fence 粒度稍小，通常在不同的 render queue 之间进行数据同步操作，用的比较多。
- Event ：比 Semaphore 粒度更小，常用在 command buffer 之间同步操作。
- Barrier ：pipeline 内的的内存访问和资源状态转移。

Nvidia 有个更加直观的图片说明最常用的情况：

![](/images/2022-09-19-vulkan_sync.png)

在 swapchain 中，我们如果要等待 GPU present 完成，就用 fence。如果我们要获取一帧图像，需要等待一个队列被提交，就用 semaphore。在主屏幕的 present 过程中，这个等待是个惯用方式，所以我们可以在 simple vulkan 中，把信号量和 swapchain 进行一个紧绑定，隐藏这个细节。



## 参考资料

无





