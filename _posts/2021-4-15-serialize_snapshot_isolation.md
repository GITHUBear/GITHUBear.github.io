---
layout: posts
title: Serializable Snapshot Isolation
excerpt: "leetcode 周赛代码模板整理"
modified: 2021-04-15
tags: [leetcode, code template]
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

## Snapshot isolation

- 读事务并不延迟
- 写事务为了避免丢失更新，采取先提交先获胜的原则

## Write skew

`SI` 的事务隔离级别，虽然具有读事务无需等待的优势，但是依然不能做到 `Serializable` ，这是因为在某些情况下，`SI` 会允许某些事务交错执行，从而导致数据库的一致性被破坏（例如某些数据存在着一致性的关联，但是 `SI` 可能会把这种一致性破坏），这种现象被称为 `Write Skew`

## 冲突图理论

