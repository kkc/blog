---
title: Paper 閱後心得： The Tail at Scale
catalog: true
comments: true
indexing: true
header-img: ../../../../img/default.jpg
top: false
tocnum: true
date: 2021-09-24 22:39:13
subtitle:
tags:
categories:
---

## 前言

最近常在看一個 youtube [Vivek Haldar](https://www.youtube.com/c/VivekHaldar) 讀 paper 順便幫忙畫重點，正好看他又翻到這篇很有名的 paper: [The Tail at Scale](https://research.google/pubs/pub40801/)，其第一作者更是大名鼎鼎的 Jeff Dean, 事隔多年再加上在這一行也幹了一陣子，其實更佩服 google 的論文裡面提到的一些在實作上面的方法，而不是只有學術上面的數據，重新看了這篇，也讓我回憶起以前為了要消除 tail latency 所做的一些努力，很推薦大家也來重讀這個經典的論文。

## Tail latency 是什麼

現在的我們都知道，建一個線上系統，網頁或是 API 的 response time 對於 customer 是非常重要的，而一個線上系統的 response time distribution 通常會長得像下面這張圖。

![](https://i.imgur.com/4j2U45h.png =600x)
Credit: https://robertovitillo.com/why-you-should-measure-tail-latencies/

Tail latency 一般我們會拿來說是 99th and 99.9th percentiles 這個區間的 requests，一般來說在 tail latency 是有機會慢到讓人髮指的，而從上面的圖來看似乎量體不那麼大，不會讓人太擔心。

![](https://i.imgur.com/7f0qYWb.png =600x)
Credit: https://robertovitillo.com/why-you-should-measure-tail-latencies/

在 paper 裡面寫到，如果單純只考慮一台 server，其 99th tail latency 是 1s，則 100 個 user requests 裡面只會有 1 個 user request 會受到影響。

> consider a system where each server typically responds in 10ms but with a 99th percentile latency of one second. If a user request is handled on just one such server, one user request in 100 will be slow (one second).

但是其實事實常常跟直覺有落差，現在的線上服務，常常一個 request，底下不知道有多少 service 被呼叫，這張圖告訴我們如果底下同時呼叫了 100 個 services，這樣的話，會有 63% 的 user requests 會超過 1s。另外一個有趣的例子是，10000 個 requests 的其中 1 個 request 比較慢的情況，如果下面有 2000 台 server，則會有將近 20% 的機率 user request 會超過 1s，所以在複雜的系統中，tail latency 反而是很需要被關照的。

> The figure here outlines how service-level latency in this hypothetical scenario is affected by very modest fractions of latency outliers. If a user request must collect responses from 100 such servers in parallel, then 63% of user requests will take more than one second (marked "x" in the figure). Even for services with only one in 10,000 requests experiencing more than one-second latencies at the single-server level, a service with 2,000 such servers will see almost one in five user requests taking more than one second (marked "o" in the figure).

![](https://i.imgur.com/TmqsB8N.png =600x)

## Google service 真實情況

下面這個表格告訴我們，一個 google 內部 service 的真實分佈，對於一個 leaf service，他的 99th latency 是 10ms, 但是通過 fanout 後，對於 95% 請求完成的 99th latency 則是 70ms，100% 則是 140ms，就此我們可以觀察到剩下的 5% 請求，幾乎製造了一半的 latency (70ms)，所以最佳化這些 slow outlier 對於整體系統會有很大的幫助。 

> waiting for the slowest 5% of the requests to complete is responsible for half of the total 99%-percentile latency. Techniques that concentrate on these slow outliers can yield dramatic reductions in overall service performance.

![](https://i.imgur.com/HLFXXq4.png)

## 產生 tail latency 的原因 (Why Variability Exists)

- 軟體方面:
  - shared resources: 在同台機器上面 CPU, memory, networking 的競爭。
  - daemon & maintaince activities: 一些 backgroud processes 不定時的被啟動執行，造成原本的 request response time 受到影響。(e.g data compaction & garbage collection)
 
- 硬體方面:
  - Power limits: CPU 被設計成可以暫時跑超過他的平均功率上，而得到效能提升，但是如果這情況發生太久就會被 throttle 降溫。
  - SSD GC: SSD 發生 GC 的時候，也會變慢

## 解決 tail latency 的方法

### Reduce Component Variability (Component 內部)

- Differentiating service classes and higher-level queuing: 將請求分類，讓一些需要 interactive 的請求能夠先被處理。
- Reducing head-of-line blocking: 長時間的請求常常會造成 tail latency，我們可以把這種請求，打散成一連串小的請求，這樣可以跟其他的小請求交互執行，讓這些小請求不會被卡在 queue 後面，造成整體的 tail latency 變差。
- Managing background activities and synchronized disruption: 將 background process 拆解成成本較低的操作，並且搭配 throttle 的功能，然後在整體 system loading 比較低的時候再去執行。 

** 這篇論文特別指出他略過 cache 的討論，因為 cache 的情況又更複雜 **

### Living with Latency Variability

- Hedged requests: 將同樣的 request 發給不同的 servers，然後拿先返回的 response 回傳給 user，一但拿到 response，就把另外未處理完的 request 取消掉，但這件事大家一定有疑問，不是會造成 server loading 加重嗎？其中論文裡面的一個方法是，只有當第一個 request 到 95th 的時間時還沒拿到 response，才再發出第二個 request，這樣的做法約略讓整體的 loading 提升 5%，但是卻可以有效地減低 tail latency。(想法：類似的做法在 map reduce 的論文裡面也可以看到，提升一點 loading 讓整體系統加快，很值得我們去思考類似的解法，下面我還列了一個論文裡面的範例)
  > For example, in a Google benchmark that reads the values for 1,000 keys stored in a BigTable table distributed across 100 different servers, sending a hedging request after a 10ms delay reduces the 99.9th-percentile latency for retrieving all 1,000 values from 1,800ms to 74ms while sending just 2% more requests.

- Tied requests: 跟 Hedged requests 的做法差不多，只是這裡是直接把 requests 發給多個 server 的 queue，而不是等待一段時間再發，server 做完後會主動取消其他的 server 的 request，不過這件事會增加網路的負擔
  > instead of delaying before sending out hedged requests, enqueue requests simulateously on multiple servers, but tie them together but telling each server who else also has this in their queue. When the first server processes the request, it tells the others to cancel it from their queues. 

### Cross-Request Long-Term Adaptations

- Micro-partition: 將一台大 server 切割成多個小 partition，類似一台機器切成 20 個 partition，這樣可以在動態任務分配(dynamic assignment)上面更靈活，主要就是讓 granularity 切得更細。
- Selectively increase replication factors: 當一個 partition 變成 hot 的時候，能夠動態增加 replica 去 share loading。(想法: 這個目前有很多 autoscaling 的技術，著重的就是這個點)
- Put slow machines on probation: 當機器變慢的時候，將他隔離起來，這也是 circuit breaker，熔斷讓 machine 的情況不會變更差。

- Consider ‘good enough’ responses: 服務降級，讓 service 還是有回應，但是給的內容也許不完整。

## Conclusion

以上就是這篇論文的閱讀筆記，其實裡面有些概念，目前都被推廣的蠻多的，像是 autoscaling, service degradation, circuit breaker, 不過還是有些其實對於一般大眾來說還不是那麼熟悉，像是 Hedged requests & Tied requests，都蠻有趣的，也被實際證實它的用處，這個方法還在 Rob Pike 的 golang concurrency talk 裡面被拿來當作範例，的確在 google 應該是耳熟能詳的技術，之後在設計多 fan out 的系統，也許可以拿來採用。


## Reference

- https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext
- https://www.youtube.com/watch?v=1Qxnrf2pW10
- https://blog.acolyer.org/2015/01/15/the-tail-at-scale/
