---
layout: posts
title: Percolator 论文理解
excerpt: "Percolator 论文总结相关"
modified: 2021-04-10
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

## Percolator

`Percolator` 在 `BigTable` 的基础上实现，相较于 `BigTable` 来说，`Percolator` 主要延伸出了如下新的特性：

- 多行事务：保证 `snapshot isolation` 隔离级别的多行事务模型
- 观察者框架：当用户指定的列发生变化时，系统调用的代码段

### 数据样式

因为 `Percolator` 基于 `BigTable` 实现，所以在数据样式或者说数据的逻辑存储结构上直接利用了 `BigTable` 的多维映射表结构。

对于每一个 `BigTable` 当中的 row，`Percolator` 将需要的元数据作为一个新的 column 组织到了 `BigTable` 当中，column 定义如下：

|  *Column*  |  *Use*  |
|:------------:|:---------:|
| **c:lock** | An uncommitted txn is writing this cell; contains the location of primary lock |
| **c:write** | Committed data present; Stores the BigTable timestamp of the data |
| **c:data** | Stores the data itself |
| **c:notify** | Hint: observers may need to run |
| **c:ack_O** | observer "O" has run; stores start timestamp of successful last run |

- `notify` 表示是否需要触发在某些列上监听的 `observers`
- `ack` 是一个简单的时间戳值，表示最近执行通知的观察者的开始时间
- `data` kv 数据，key是时间戳，value是真实数据，包含多个entry
- `write` KV结构，key是事务 `commit` 时间戳，value是各个时间戳下曾经写入的值
- `lock` KV结构，key是事务 `start` 时间戳，value是锁的内容

### Percolator 事务

#### 目标

- 锁必须持久化防止一个锁在两阶段提交之间消失，导致提交冲突的事务 --> **冗余备份**
- 锁服务需要高吞吐量 --> **分布式**
- 锁服务需要低延迟 --> **负载均衡**

而以 `BigTable` 为基础，以上几点基本可以满足，所以在 `percolator` 中的实现就是将锁和数据存储在同一行，锁成为了一个特殊的数据列，`percolator` 在一个 `BigTable` 的行事务中对锁列进行读取和修改。

#### Txn code

```c++
class Transaction {
    struct Write{ Row row; Column: col; string value;};
    vector<Write> writes_;
    int start_ts_;

    Transaction():start_ts_(orcle.GetTimestamp()) {}
    void Set(Write w) {writes_.push_back(w);}
    bool Get(Row row, Column c, string* value) {
        while(true) {
            bigtable::Txn = bigtable::StartRowTransaction(row);
            // Check for locks that signal concurrent writes.
            if (T.Read(row, c+"locks", [0, start_ts_])) {
                // There is a pending lock; try to clean it and wait
                BackoffAndMaybeCleanupLock(row, c);
                continue;
            }
        }

        // Find the latest write below our start_timestamp.
        latest_write = T.Read(row, c+"write", [0, start_ts_]);
        if(!latest_write.found()) return false; // no data
        int data_ts = latest_write.start_timestamp();
        *value = T.Read(row, c+"data", [data_ts, data_ts]);
        return true;
    }
    // prewrite tries to lock cell w, returning false in case of conflict.
    bool Prewrite(Write w, Write primary) {
        Column c = w.col;
        bigtable::Txn T = bigtable::StartRowTransaction(w.row);

        // abort on writes after our start stimestamp ...
        if (T.Read(w.row, c+"write", [start_ts_, max])) return false;
        // ... or locks at any timestamp.
        if (T.Read(w.row, c+"lock", [0, max])) return false;

        T.Write(w.row, c+"data", start_ts_, w.value);
        T.Write(w.row, c+"lock", start_ts_, 
            {primary.row, primary.col});  // The primary's location.
        return T.Commit();
    }
    bool Commit() {
        Write primary = write_[0];
        vector<Write> secondaries(write_.begin() + 1, write_.end());
        if (!Prewrite(primary, primary)) return false;
        for (Write w : secondaries)
            if (!Prewrite(w, primary)) return false;

        int commit_ts = orcle.GetTimestamp();

        // Commit primary first.
        Write p = primary;
        bigtable::Txn T = bigtable::StartRowTransaction(p.row);
        if (!T.Read(p.row, p.col+"lock", [start_ts_, start_ts_]))
            return false; // aborted while working
        T.Write(p.row, p.col+"write", commit_ts,
            start_ts_); // Pointer to data written at start_ts_
        T.Erase(p.row, p.col+"lock", commit_ts);
        if(!T.Commit()) return false;  // commit point

        // Second phase: write our write records for secondary cells.
        for (Write w:secondaries) {
            bigtable::write(w.row, w.col+"write", commit_ts, start_ts_);
            bigtable::Erase(w.row, w.col+"lock", commit_ts);
        }
        return true;
    }
}; // class Transaction
```