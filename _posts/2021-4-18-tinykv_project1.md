---
layout: posts
title: TinyKV-Project1
excerpt: "tinykv 实验报告"
modified: 2021-04-15
tags: [pingcap, tinykv]
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

## 目标

- 为 `StandAloneStorage` 实现 `Storage` 接口
- 为 `Server` 结构实现 raw `Put/Get/Delete/Scan` 方法

## 流程

不管三七二十一，先执行 `make project1` 看看要完成 project1 需要通过哪个测试文件，可以定位到 `kv/server/server_test.go`，首先进行测试的是 `TestRawGet1`

```go
func TestRawGet1(t *testing.T) {
	conf := config.NewTestConfig()
	s := standalone_storage.NewStandAloneStorage(conf)
	s.Start()
	server := NewServer(s)
	defer cleanUpTestData(conf)
	defer s.Stop()

	cf := engine_util.CfDefault
	Set(s, cf, []byte{99}, []byte{42})

	req := &kvrpcpb.RawGetRequest{
		Key: []byte{99},
		Cf:  cf,
	}
	resp, err := server.RawGet(nil, req)
	assert.Nil(t, err)
	assert.Equal(t, []byte{42}, resp.Value)
}
```

首先执行 `NewStandAloneStorage` 将 config 传给 `StandAloneStorage` 