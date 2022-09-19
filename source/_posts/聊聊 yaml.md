---
title: 聊聊 yaml
date: 2022年9月19日
tags: engine
---

最近确定了引擎的场景文件的格式为 yaml，配置文件也会考虑使用 yaml，不过仍然会保留对 scl 内部的简单 ini 文件的支持。所以想聊聊为什么做出这个选择，以及 yaml 的优势和劣势。

<!-- more -->

cat 渲染器的场景结构需要保存为一个文件格式，我希望这个格式可以满足如下的一些点：

- 支持层级结构
- 人类可读，并易于手动编辑
- 足够紧凑
- 有健壮的解析器

之前在做 cat ui 的时候，ui 的窗口文件也需要类似这样一个文件格式，当时我的选择不多，只有 json 和 xml。由于受到魔兽世界的 ui  wow interface 的影响，也选择了 xml 作为窗口的描述文件，并且还有一个描述窗口加载顺序的 toc 类型的自定义文件。

现在看来，魔兽的这些选择可能基于十几年前并不丰富的数据格式经验。随着行业发展，尤其是 web 的开发对更精巧可用的文件格式的需求，推动了数据格式的创新。

下面是 wiki 上一个目前几乎所有的序列化文件格式的列表和特性对比：

[Comparison of data-serialization formats](https://en.wikipedia.org/wiki/Comparison_of_data-serialization_formats)

作为对 xml 的不足的一些补充，业界最常用和成熟的 yaml 和 json 进入了我的视野。yaml 是因为 unity 的广泛使用被我熟知，但是其实 yaml 在 web 的很多领域都有大量的使用。json 就不用说了，发展的很成熟了已经。所以我最终要在这三者之间找到最适合我的数据格式。

其实选择原因也不难，google 一下 yaml vs json，或者 xml vs json，xml vs yaml，能找到一大票的文章来谈优缺点，xml 对我而言不够简洁，而且在复杂项目中，也很难用手配置，所以首先出局了。json 的大量括号也不够紧凑，所以我最终更倾向于 yaml。

在选定了 yaml 之后，我发现不少人提到了他和 toml 的对比，toml 是一个类似 ini 的文件格式，也符合一些条件，但是当树形结构复杂之后，会存在不少问题。

当然，yaml 和 toml 以及 json 相比，有个致命的缺陷，就是解析器太复杂太难以实现了。并且，从实施的角度看，yaml 官网推荐的几个 c/c++ 解析器都不大好用，基本上和我们常用的 xml 或者 json 的解析器的 api 设计差别很大，概念上也难于理解。

下面是官方的几个解析库：

```yaml
YAML Frameworks and Tools:
  C/C++:
  - libfyaml      # "C" YAML 1.2 processor (YTS)
  - libyaml       # "C" Fast YAML 1.1 (YTS)
  - libcyaml      # YAML de/serialization of C data (using libyaml)
  - yaml-cpp      # C++ YAML 1.2 implementation
```

我最终在尝试了几天之后，放弃了这些解析器，转而使用一个不太成熟但是更易用的解析器：[rapidyaml](https://github.com/biojppm/rapidyaml)

这个解析器有着和 pugixml 类似的友好的接口设计，并且内部没有使用 STL，所以无需担心内存碎片和头文件膨胀问题。

当然，打开代码会发现，这个项目还没有完成，属于基本可用，但是扩展功能不完善的状态，这个项目是2020年左右的项目，第一个 release 是2020年的11月3日发布的，可能成熟的版本是2021年才发布，所以还不到一年时间，需要再等待作者开发一段时间稳定下来。

最后，我们看看作者写在项目的 readme 的一段话，就能体会到寻找一个好用的 yaml 的 c/c++ 解析器有多难了：

```
Alternative libraries
Why this library? Because none of the existing libraries was quite what I wanted. When I started this project in 2018, I was aware of these two alternative C/C++ libraries:

libyaml. This is a bare C library. It does not create a representation of the data tree, so I don't see it as practical. My initial idea was to wrap parsing and emitting around libyaml's convenient event handling, but to my surprise I found out it makes heavy use of allocations and string duplications when parsing. I briefly pondered on sending PRs to reduce these allocation needs, but not having a permanent tree to store the parsed data was too much of a downside.
yaml-cpp. This library may be full of functionality, but is heavy on the use of node-pointer-based structures like std::map, allocations, string copies, polymorphism and slow C++ stream serializations. This is generally a sure way of making your code slower, and strong evidence of this can be seen in the benchmark results above.
Recently libfyaml appeared. This is a newer C library, fully conformant to the YAML standard with an amazing 100% success in the test suite; it also offers the tree as a data structure. As a downside, it does not work in Windows, and it is also multiple times slower parsing and emitting.

When performance and low latency are important, using contiguous structures for better cache behavior and to prevent the library from trampling caches, parsing in place and using non-owning strings is of central importance. Hence this Rapid YAML library which, with minimal compromise, bridges the gap from efficiency to usability. This library takes inspiration from RapidJSON and RapidXML.
```

大意是：

```
为啥我开发这个 yaml 库？因为现有的 yaml 库都不好用。项目从2018年开始开发，最初我关注了两个库：

libyaml：一个纯c库，paring 和 emmiting 都是基于 event 机制，难于理解，没有一个树形的数据结构表达。并且内存策略糟糕，大量字符串内存拷贝。

yaml-cpp: 有树形数据表达，能满足需求，但是重度使用 std::map,字符串拷贝等内存使用也很随意，还用了效率低下的 c++ stream。

最近还有个库 libfyaml，这是一个新的 yaml 解析库，对 yaml 标准的支持非常的好，能达到100%通过官方 test suit。也有一个树形的数据结构表达。
但是这个库完全不支持 Windows。

当性能和延迟比较重要的时候，我们需要考虑cache效率，希望 rapidyaml 能达到 rapidJson 和 rapidxml 的水平。
```

其中无奈，可见一斑  ┐(—__—)┌
