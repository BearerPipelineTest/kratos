# 自适应限流保护

kratos 借鉴了 Sentinel 项目的自适应限流系统，通过综合分析服务的 cpu 使用率、请求成功的 qps 和请求成功的 rt 来做自适应限流保护。


## 核心目标

* 自动嗅探负载和 qps，减少人工配置
* 削顶，保证超载时系统不被拖垮，并能以高水位 qps 继续运行


## 限流规则

1，指标介绍

|指标名称|指标含义|
|---|---|
|cpu|最近 1s 的 CPU 使用率均值，使用滑动平均计算，采样周期是 250ms|
|inflight|当前处理中正在处理的请求数量|
|pass|请求处理成功的量|
|rt|请求成功的响应耗时|


2，滑动窗口

在自适应限流保护中，采集到的指标的时效性非常强，系统只需要采集最近一小段时间内的 qps、rt 即可，对于较老的数据，会自动丢弃。为了实现这个效果，kratos 使用了滑动窗口来保存采样数据。

![ratelimit-rolling-window](/doc/img/ratelimit-rolling-window.png)

如上图，展示了一个具有两个桶（bucket）的滑动窗口（rolling window）。整个滑动窗口用来保存最近 1s 的采样数据，每个小的桶用来保存 500ms 的采样数据。
当时间流动之后，过期的桶会自动被新桶的数据覆盖掉，在图中，在 1000-1500ms 时，bucket 1 的数据因为过期而被丢弃，之后 bucket 3 的数据填到了窗口的头部。


3，限流公式

判断是否丢弃当前请求的算法如下：

`cpu > 800 AND (Now - PrevDrop) < 1s AND (MaxPass * MinRt * windows / 1000) < InFlight`

MaxPass 表示最近 5s 内，单个采样窗口中最大的请求数。
MinRt 表示最近 5s 内，单个采样窗口中最小的响应时间。
windows 表示一秒内采样窗口的数量，默认配置中是 5s 50 个采样，那么 windows 的值为 10。

## 压测报告

场景1，请求以每秒增加1个的速度不停上升，压测效果如下：

![ratelimit-benchmark-up-1](/doc/img/ratelimit-benchmark-up-1.png)

左测是没有限流的压测效果，右侧是带限流的压测效果。
可以看到，没有限流的场景里，系统在 700qps 时开始抖动，在 1k qps 时被拖垮，几乎没有新的请求能被放行，然而在使用限流之后，系统请求能够稳定在 600 qps 左右，rt 没有暴增，服务也没有被打垮，可见，限流有效的保护了服务。


参考资料：

[Sentinel 系统自适应限流](https://github.com/alibaba/Sentinel/wiki/%E7%B3%BB%E7%BB%9F%E8%87%AA%E9%80%82%E5%BA%94%E9%99%90%E6%B5%81)

-------------

[文档目录树](summary.md)