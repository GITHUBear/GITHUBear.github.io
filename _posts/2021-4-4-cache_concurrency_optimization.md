---
layout: posts
title: Cache Optimization
excerpt: "cache 优化相关技术"
modified: 2021-04-04
tags: [cache]
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

## Optimization Guideline

- The critical observation to make is that **the user of the cache does not care what the current LRU ordering is.**

- The only concern of the caller is that **the cache maintains a threshold size and a high hit rate.**

## LRU Approach

- capacity-based

- time-based

- reference-based

## Example

- memcache

## ConcurrentLinkedHashMap

### Cacheline状态

- active, kept in map and queue

- retire, deleted from map but not from queue

- dead, deleted from map and queue

