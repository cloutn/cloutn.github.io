---
title: Houdini 的 HDA 保存为文件夹
date: 2022年8月11日
tags: houdini
---

# Houdini 的 HDA 保存为文件夹

Houdini 的 HDA 是一个单一的二进制文件，不方便 svn 和 git 这样的版本管理工具管理，多人同时编辑的时候也容易冲突。因此，Houdini 本身提供了一个 hotl 工具来将 HDA 转换为一个文件夹，文件夹里面保存的大多是文本格式的文件。

<!-- more -->

转换方法如下：

在 Houdini 的菜单中，选择 **Assets ▸ Asset Manager**. 右键点击想要转换的 HDA 文件，选择 **Convert to unpacked format** (或者 **Convert to packed format**).

转换是通过 hotl.exe 这个工具实现的，我们也可以自己调用这个工具完成 HDA 单个文件到目录之间的相互转换。

``` apl
hotl -x|-X|-t <directory> <hdafile>  	# 从 HDA 单个文件转为目录
hotl -C|-C|-l [-b] <directory> <hdafile>	# 从 HDA 目录转为单个文件
```

在菜单中的选项用的就是 -t 的选项。

具体详细用法可以参考 Houdini 官方的文档：

[hotl utility](https://www.sidefx.com/docs/houdini/ref/utils/hotl.html)
