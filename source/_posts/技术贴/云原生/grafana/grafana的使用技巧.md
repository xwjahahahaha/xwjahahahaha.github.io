---
title: grafana的使用技巧
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-07-22 13:59:13
---

[TOC]


<!-- more -->

> 参考：
>
> * https://grafana.com/docs/grafana/latest/variables/variable-types/global-variables/

# 一、全局变量

官方相关资料：https://grafana.com/docs/grafana/latest/variables/variable-types/global-variables/

## $__interval

You can use the `$__interval` variable as a parameter to group by time (for InfluxDB, MySQL, Postgres, MSSQL), Date histogram interval (for Elasticsearch), or as a *summarize* function parameter (for Graphite).

使用此变量作为时间间隔对数据分组

## $__rate_interval

Currently only supported for Prometheus data sources. The `$__rate_interval` variable is meant to be **used in the rate function**. Refer to [Prometheus query variables](https://grafana.com/docs/grafana/latest/datasources/prometheus/) for details.

简单理解就是一个prometheus提供的一个升级版的`$__interval`，它能够更好的计算时间间隔，以免出现当缩小到过小时造成没有数据

它的计算方式为：`max( $__interval + Scrape interval, 4 * Scrape interval)`

where Scrape interval is the Min step setting (AKA queryinterval, a setting per PromQL query)

更多详细可见：https://grafana.com/blog/2020/09/28/new-in-grafana-7.2-__rate_interval-for-prometheus-rate-queries-that-just-work/

# 二、计算的操作

## increase(<expr>[$__rate_interval])

**Calculates the increase in the time series in the range vector.** Breaks in monotonicity (such as counter resets due to target restarts) are automatically adjusted for. The increase is extrapolated to cover the full time range as specified in the range vector selector, so that it is possible to get a non-integer result even if a counter increases only by integer increments.

* 作用：计算整个时间序列范围内的增量
* 增量会被平摊到/覆盖到整个时间序列，所以即使是增量是整数也可能会被计算为一个小数

## rate(<expr>[$__rate_interval])

Calculates the per-second average rate of increase of the time series in the range vector. Breaks in monotonicity (such as counter resets due to target restarts) are automatically adjusted for. Also, the calculation extrapolates to the ends of the time range, allowing for missed scrapes or imperfect alignment of scrape cycles with the range's time period.

* 计算范围向量中时间序列的每秒平均增长率
* 此外，此计算函数可以推断到时间范围的末端，考虑到遗漏的数据抓取或者数据抓取时间范围与时间范围的不完美对齐的情况
