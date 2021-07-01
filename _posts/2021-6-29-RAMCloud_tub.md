---
layout: posts
title: RAMCloud-Tub
excerpt: "RAMCloud源码阅读"
modified: 2021-06-29
tags: [RAMCloud, distribute system]
comments: true
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

## Tub

> A Tub holds an object that may be uninitialized; it allows the allocation of memory for objects to be separated from its construction and destruction. When you initially create a Tub its object is uninitialized (and should not be used). You can call #construct and #destroy to invoke the constructor and destructor of the embedded object, and #get or #operator-> will return the embedded object. The embedded object is automatically destroyed when the Tub is destroyed (if it was ever constructed in the first place).

`Tub` 将对象的构造和析构与对象的内存空间分配分离，存在于 `Tub` 中的对象在一开始时是没有初始化的，除非调用 `Tub` 的 `construct` 以及 `destroy` 方法，总之就是一个对于对象的包装，这种延后构造的方式能在以下的场景中带来好处：

- 需要创建一个 `object arrary`，但是对象的构造函数处理代价很高，直到需要时再调用对象的构造器能够有效地提高性能；
- 起到类似 `std::unique_ptr` 的作用，destruction guard；
- 提供类似 `Rust` 的 Option<> 对象，Some or None
- 支持 `singleton`，且无需手动管理生命周期

### 成员

```c++
ElementType object[0];
char raw[sizeof(ElementType)];
bool occupied;
```

`object` 初始化完成之后的对象，即 `raw` 中的空间地址（也就是 raw 中保存了初始化后的对象数据）；`occupied` 记录对象是否初始化。

