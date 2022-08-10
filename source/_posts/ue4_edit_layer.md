---
title: UE4 中的 Edit Layer 原理
date: {{ date }}
tags:
- ue4
comments: true
---

在 ue4 中，地形 paint 的时候可以选择 material 上设置好的 layer，如下所示：

![](/images/2022-08-10-17-26-27-image.png)

<!-- more -->

按照文档可以对 landscape 开启 Edit Layer

[Landscape Edit Layers | Unreal Engine Documentation](https://docs.unrealengine.com/4.27/en-US/BuildingWorlds/Landscape/Layers/)

![](/images/2022-08-10-17-28-05-image.png)



开启后，在对地形 paint 的时候，可以先选择使用的 target layer，同时还可以指定当前 brush 使用哪一个 Edit Layer

![](/images/2022-08-10-17-30-17-image.png)



当我们将 Edit Layer 设置为 visible 或者 hide 的时候，其所使用的 brush 的内容也会被取消掉。



在代码中，Edit Layer 对应的类是

```cpp
struct FLandscapeLayer
{
    ...
	TArray<FLandscapeLayerBrush> Brushes;    
    ...
}
```

编辑时 brush 信息储存在 Brushes 变量中。

class ALandscape 类中有一个 FLandscapeLayer 数组，就是为了保存 Edit Layer 刷子信息的。

```cpp
class ALandscape : public ALandscapeProxy
{
...
    TArray<FLandscapeLayer> LandscapeLayers;
...
}
```

当我们修改了 Edit Layer 的属性后，比如 set visible 或者刷了新的数据，会触发

```cpp
int32 ALandscape::RegenerateLayersWeightmaps(...)
```

在这个函数中，会用 Computer Shader 将 brush 中的数据拷贝到实际的 Target Layer 的数据中。

```cpp
PrepareComponentDataToExtractMaterialLayersCS(
    InLandscapeComponentsToRender, 
    Layer, 
    CurrentWeightmapToProcessIndex, 
    LandscapeExtent.Min, 
    OutputDebugName, 
    WeightmapScratchExtractLayerTextureResource, 
    ComponentsData, 
    LayerInfoObjects);

```



从 Edit Layer 中读取数据的地方就在这个函数的这里：

```cpp
	for (ULandscapeComponent* Component : InLandscapeComponents)
	{
		FLandscapeLayerComponentData* ComponentLayerData = Component->GetLayerData(InLayer.Guid);
		if (ComponentLayerData != nullptr)
		{
			if (ComponentLayerData->WeightmapData.Textures.IsValidIndex(InCurrentWeightmapToProcessIndex) && ComponentLayerData->WeightmapData.TextureUsages.IsValidIndex(InCurrentWeightmapToProcessIndex))
			{
				UTexture2D* LayerWeightmap = ComponentLayerData->WeightmapData.Textures[InCurrentWeightmapToProcessIndex];
```

LayerWeightmap 就是在 Edit Layer 中刷的数据。
