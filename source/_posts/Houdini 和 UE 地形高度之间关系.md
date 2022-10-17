---
title: Houdini 和 UE 地形高度之间关系
date: 2022年10月17日
tags: tool
---

## 背景

当我们使用 Houdini 和 UE 制作地形的时候，经常需要把 UE 中的 Landscape 数据导入为 Houdini 中的 Height Field 数据。这个双向的过程往往通过 Houdini Engine for Unreal 插件就能很好的完成。

但是某些情况下，我们需要从 UE 导出高度图到其他某个软件（如GAEA）中进行处理，处理后再导出为高度图导入到 Houdini 中，这个过程，就需要我们从 UE 导出为高度图 PNG 文件，再从 PNG 文件导入到 Houdini 中。如果 XZ 尺寸严格按照 UE 和 Houdini 文档设定，往往是没有问题的，但是高度在 UE 和 Houdini 中的定义是不同的，这就要求我们了解 UE 和 Houdini 导出到高度图的时候，具体是如何处理高度的。

<!-- more -->

## 机制

首先说下 UE 是如何处理高度图的导入导出的。

UE 中的高度图导出的时候，是将高度值写入到一个 16 bit 的图片中，png 或者 raw 格式都可以。也就是说，其值的范围是 0 ~ 65525。在其内部存储中，高度图是作为一个纹理保存的，当我们要导出的时候，是读取该纹理的值，并通过如下方式将 RG 两个通道转为一个 16 bit 的高度值的。如下所示（LandscapeEditInterface.cpp:661）：

```c++
FColor& TexData = HeightmapTextureData[ TexX + TexY * SizeU ];
uint16 Height = (((uint16)TexData.R) << 8) | TexData.G;
StoreData.Store(LandscapeX, LandscapeY, Height);
```

也就是说，存储高度图的纹理保存的值也是 0 到 65535 的一个值。直到渲染的时候，才会把这个值映射到实际世界的 -256 米到 256 米的高度范围内。

然后我们说下 Houdini 中的 Height Field 是如何处理高度图的导出的。

Houdini 中导出的时候，会自动计算一个地形的最高点和最低点，然后将最低点保存为0，最高点保存为1。输出的通道和格式都是可控的，后面我们会看到。我们也可以手动设置最高点1和最低点0对应的高度值。所以我们就可以从这里入手，让 Houdini 去适应 UE 的 -256 到 256 的高度范围。

## 实现

下面分几种情况讨论导入导出的方法：

- UE 导出为 PNG
- PNG 导入到 UE
- Houdini 导出为 PNG
- PNG 导入到 Houdini

1. UE 导出为 PNG

   这个很简单，如下所示：

​	![](/images/2022-10-17-1.png)

2. PNG 导入到 UE

   在创建地形的时候，直接指定 PNG 文件即可，如下所示：

​	![](/images/2022-10-17-2.png)

3. Houdini 导出为 PNG

   使用 HeightField Output 节点，设置如下：

   Format 选择 Single Channel

   Type 选择 16b Fixed

   image Channel 中的 Red 设置为 Height

   Output Range 设置为 Manual Remap

   From Range 设置为 -256 到 256。（由于 UE 中的高度图范围是 -256 到 256）

   To Range 设置为 0 到 1.

4. PNG 导入到 Houdini

   首先使用一个 HeightField file 节点，并设置如下：

   Size Method 设置为 Grid Spacing

   Grid Spacing 设置为 1

   Height Scale 设置为 512.

   然后再连接一个 HeightField_xform 节点，并设置如下：

   Height Offset 设置为 -256

