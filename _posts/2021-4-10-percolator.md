---
layout: posts
title: Docs for MeanShift & GMM
excerpt: "Summary of relevant materials and documents of MeanShift and GMM"
modified: 2021-03-29
tags: [bigtable, distribute system, google]
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

## BigTable

> `percolator` 的设计基于 `BigTable` 实现，故在此简单介绍 `BigTable` 的架构，详细可参考 [论文-中文译](https://arthurchiao.art/blog/google-bigtable-zh/)

### 数据存储的逻辑格式

一个 `Bigtable` 就是一个稀疏、分布式、持久的多维有序映射表(map)，数据通过行键、列键和一个时间戳进行索引，表中的每个数据项都是不作理解(指 `Bigtable` 本身不存储数据的任何语义)的字节数组。

> 映射关系：(row:string, column:string, time:int64) -> string

![table_data_scheme](../image/posts/2021-4-10-percolator/2021-04-10-percolator_bigtable_data_scheme.png)

以 `Google` 的 `Webtable` 为例：

- row: URL
- columns: contents: 网页内容
- column family: anchor，网页链接锚点文本
- ti: 时间戳

### 基本架构

![bigtable_overview](../image/posts/2021-4-10-percolator/2021-04-10-percolator_bigtable_overview.png)

在 `BigTable` 系统中的主要组件以及概念：

- master server(负责负载均衡): 

    1. 负责新 `tablet` 的分配
    2. 监控 `tablet server` 的状态
    3. 处理来自 `tablet server` 的 `split` 请求

- tablet server:

    1. 处理来自 `client` 的读写请求
    2. 发起 `tablet` 的分裂与合并

- Chubby:

    1. 在 `master server` 的启动阶段，通过分布式锁选出一个唯一的 `master server`
    2. 可用于查找某个 `tablet` 的位置(哪个 `tablet server` 上分配了指定的 `tablet`)
    3. 访问控制元数据
    4. 帮助 `master server` 跟踪 `tablet server`，诸如发现新 `server` 以及挂掉的 `server` 等等，集群可扩展

