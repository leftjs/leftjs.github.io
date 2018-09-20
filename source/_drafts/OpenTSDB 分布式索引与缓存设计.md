---
title: OpenTSDB分布式索引与缓存设计
date: 2018-09-20 11:48:37
tags:
  - 索引
  - 分布式
categories: 数据库
---

# 提出背景

1. 缓解 OpenTSDB 对 HBase 直接写入时的压力,提高 OpenTSDB 的吞吐能力
2. 时间范围查询需要扫描全盘的 rowkey，并且缺少多维过滤和聚合能力

# 设计方案

以 Golang 编写高效 proxy 的方式提供分布式缓存与分布式索引能力应对以上问题

# 参考文献

> 1. Influxdb 原理详解: [http://hbasefly.com/2018/01/13/timeseries-database-4/](http://hbasefly.com/2018/01/13/timeseries-database-4/)
> 2. TLV: [https://blog.csdn.net/chexlong/article/details/6974201](https://blog.csdn.net/chexlong/article/details/6974201)
> 3. mmp: [https://zhuanlan.zhihu.com/p/27679281](https://zhuanlan.zhihu.com/p/27679281)
> 4. TableStore 时序数据存储 - 架构篇 [https://yq.aliyun.com/articles/620991](https://yq.aliyun.com/articles/620991)
> 5. 時序數據庫技術體系 – InfluxDB 多維查詢之倒排索引 [https://hk.saowen.com/a/af78bf37ee48d525359b9195f8cf1538d85b26b05a2339febc25b2460a08c6b8](https://hk.saowen.com/a/af78bf37ee48d525359b9195f8cf1538d85b26b05a2339febc25b2460a08c6b8)
> 6. 饿了么轻量级分布式时序数据库的设计与探索: [https://dbaplus.cn/news-159-2132-1.html](https://dbaplus.cn/news-159-2132-1.html)
