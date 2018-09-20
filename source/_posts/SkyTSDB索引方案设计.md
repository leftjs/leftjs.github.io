---
title: SkyTSDB索引方案设计
date: 2018-09-20 11:48:37
tags:
  - 索引
categories: 数据库
---

# 背景

OpenTSDB 在 Tags 检索上做了不少的查询优化，但是受限于 HBase 的功能和数据模型，所以效果并不好。

# 构想

使用 golang 实现高效的读写缓存与内存索引中间件，对上提供高效的 grpc 调用接口，对下利用 native go hbase client 将 level 0 级别 datapoints
