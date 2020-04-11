---
title: 有關 Cache 的一些筆記
catalog: true
date: 2020-03-27 11:58:07
subtitle: cache is hard
header-img: cache.jpeg
tags:
- AWS
- backend
- redis
---


# 前言

最近看了 Amazon 的一篇文章 Caching challenges and strategies，在談論 cache 的種類，還有一些使用的邏輯和策略，剛好就想稍微整理一下有關於 cache 在分散式系統上面的一些筆記，這中間如果還有看到其他內容，還會再把它補起來，這邊強烈推薦大家看一下[AWS原文](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)，還有筆記最後整理的一些 Reference，相信看完大家都可以學習到很多東西。

# 何時使用 Cache

很多時候，我們會考慮加入的 Cache 的情況，不外乎就是想要加速系統的反應，或是降低 Backend 或是 Database 的負擔。而加入 Cache 往往也是一種挑戰，因為對於整個系統來說，每加入一個環節，其實都帶來了風險和複雜度，像是如果 Cache 整體 hit rate 不高，加入 Cache 也許只是讓 latency 更高，在伴隨著帶來的 Cache Availability, Cache Coherence 和 Cache Invalidation 的問題，都是我們要考慮有沒有必要使用 Cache 的關鍵。

# Cache 的種類

1. Local cache

    簡而言之，就是使用系統上面的 Memory 作為 Cache，好處是隨手可得且複雜度低，但壞處是不同機器間的 Cache consistency 的問題，另外剛開機的時候也會有 cold start 的問題。

2. External cache

    External cache 相信大家最熟的就是使用 Memcache & Redis，這個可以解決機器間 Cache consistency 的問題，但還是有可能因為更新快取的方式失敗或錯誤，造成其他 consistency 的問題，另外是加入這個元件，就提升了系統複雜度，需要有另外的機制去監控和管理，還有 availibility 也是需要考量的點，在 exteranl cache 爆炸的時候，application 需要有能力去克服這個情況。

# 常見的 external cache 的問題

這邊記錄下常見的 external cache 的問題還有解法


## Caching Penetration

如果同時間有大量的 requests 打到系統上，但是 cache 裡面沒有相對應的 key，這時候壓力就會全部灌在後端系統上(很可能是 database)，讓系統變得不穩定。

解決方案:
1. cache empty data:
    對於空的資料，也把他們 cache 起來，一般來說只存 cache key 應該佔用的空間不大。

2. bloom filter:
    利用 [bloom filter](https://www.evanlin.com/BloomFilter/) 把一定不可能存在的 request filter 掉，減少 cache penetration 的機會。

3. observe access pattern:

    可以觀察下進來的 request pattern，要對 key 設定一些規範，如果 key 長的樣子不太對，就直接 filter 掉，或是看使用者是不是來掃 database 的，對於那種長期查詢不同值的 pattern 要有防範的方法。

## Cache avalanche

Cache 雪崩，這通常發生在 cache 重啟當機，或是有大量的 cache 同時失效，此時，有大量的 requests 打進來落在 backend service 或是 database 上，如果 database 被打掛了，很有可能也沒辦法再開起來，因為會一直被大流量沖垮。

順帶一提，有時候會有大量 cache 同時失效，有可能是因為在 cache 開起來時，過期時間 (TTL) 都設定太過接近。

解決方案:
1. Cache High Avalability
    第一個想法就是確保 Cache 元件也符合 HA，像是 redis 可以使用 cluster 的模式，避免 Cache 的單點當機造成的雪崩

2. Hystrix (circuit breaker, rate limit)
    這個解法是為了保護，當 Cache 大規模失效的時候，後端的壓力會得太巨大， 像是資料庫這種東西絕對不能讓他被沖垮，所以可以出動像是 Hystrix 之類的 rate limiter 元件做*降級*處理，只讓部分的 requests 流到後面去。

3. Expiry with different TTL
    讓 key 的 TTL 都盡量分散，可以減少同時並發打到 database 的壓力。

以上的解法很可能需要混搭才是好的解法

## Cache Stampede (Thundering Herd problem)

在 cache 裡面的某個 key，經常被大量存取，屬於 cache 的 hotspot，在 *cache miss* 的時候，requests 也會一口氣打到後端或是 database 上，這個也是屬於 cache invalidation 怎麼做的範疇。

舉個例子，像是某個熱門商品的 metadata 被放在 cache 上面，cache 失效時，如果同時有 1000 個人同時 request 這個產品，這些 request 就可能會一口氣打到 database 上面，讓 database 被衝垮。

解決方案:
    基本上的解法都可以在這篇 [What is Cache Stampede](https://medium.com/@vaibhav0109/cache-stampede-problem-5eba782a1a4f) 找到

1. **Mutex (Locking)**

    有些文章會使用 request coalescing 這個詞，這邊達成這個的手段的方式就是利用 lock，讓同時間只有一個 request 可以存取 database 去更新 cache，使用 redis 可以用 setNX 或是 [distributed lock](https://redis.io/topics/distlock) 去產生這個鎖，但是使用 lock 時也要注意解鎖之類的問題。

2. **External Computation**

    使用外部的計算單元，像是用 cronjob 或是 worker + queue 的模式去更新 cache，來處理 cache invalidation 的問題，像是利用 worker 定期去掃 database 的表去更新 cache，或是利用 queue 去 trigger 更新，不過要注意如果類似掃 database 去更新的模式，有可能會存了很多不需要的資料在 cache 裡面。

3. **XFetch**

    這邊要提供第三種方法是出自一篇論文叫做 [Optimal Probabilistic Cache Stampede Prevention](http://www.vldb.org/pvldb/vol8/p886-vattani.pdf)，還有一份 slides [redisconf17-internet-archive-preventing-cache-stampede-with-redis-and-xfetch](https://www.slideshare.net/RedisLabs/redisconf17-internet-archive-preventing-cache-stampede-with-redis-and-xfetch) 講解，其實核心概念很簡單，就是在 cache 還沒過期前，提前讓*一個 worker* 去計算更新值和TTL，這個方法會那麼高效，是因為不像方法一需要引入一個 lock。

    網路上也可以找不同語言的實作:

    - golang(https://github.com/Onefootball/xfetch-go)
    - rust(https://docs.rs/xfetch/1.0.0/xfetch/)
    - ruby(https://github.com/Kache/xfetch)


# 更新 Cache 的正確姿勢

這也是一件常常被做錯的事情，但是還好網路上真的蠻多不錯的資料，我這邊就直接引用這篇[缓存更新的套路](https://coolshell.cn/articles/17416.html)，來學習下正確的方式。

考慮到下面這個更新 cache 的策略，非常的直覺，就是沒東西在 Cache 的時候，就跑去 DB 拿資料再去更新 Cache。
```
ttl = 60
val := cache.Get("key")
if val == nil {
    val := db.Read("key")
    cache.Set("key", val, ttl)
}
return val
```

乍看之下沒什麼問題，但是在並發情況下，有可能會有出乎意料之外的後果。

![](https://i.imgur.com/a92yIer.png)


從這張圖可以看到最後 Cache 裡面拿到的也許是髒資料(dirty cache data)，所以在 CoolShell 的文章裡面提到 Facebook 的論文 [《Scaling Memcache at Facebook》](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)提出的方法是，在寫的時候去把 cache 裡面的值刪掉，然後靠讀的人去更新，雖然還是有可能會發生 dirty cache data，但是在不傷害效能的情況下，已經讓發生的機率下降許多。

## 其他
- 要觀察 Data 的生命週期，去調整 TTL，如果有些 Data 不太會更新，但是又經常被存取，可以把 TTL 調高一點，而反之亦然。
- 更新不頻繁，透過鎖讓 reader 去更新 cache 資料
- 更新頻繁，可以考慮透過 queue + worker 的方式去刷新 cache。

# 結語

其實寫這篇文章是幫助我刷新一下記憶，也是把書籤的文章都看過一輪，一般來說我們的 cache 部署通常是很多層的，類似從 browser -> CDN -> local cache -> exteranl cache -> database，所以很多地方都需要這些知識，而 cache coherence 不管在硬體軟體上面都是很經典的問題，了解這些可以幫助我們更好的去架構系統。

另外在過程中也查到了 Java 有 `Ehcache`，然後 `Memcache` 的作者也用 Go 寫了一套 `Groupcache`，都是可以利用本機上面的 Memory 來達成 Distributed cache 的功能，也是蠻推薦大家去看看學習，之後應該會來爬一下 `Groupcache` 的代碼。

# Reference
- [system-design-primer#cache](https://github.com/PegasusWang/system-design-primer#cache)
- https://medium.com/@mena.meseha/3-major-problems-and-solutions-in-the-cache-world-155ecae41d4f
- http://lanlingzi.cn/post/technical/2018/0624_cache_design/
- https://www.zhihu.com/question/39114188
- https://docs.rs/xfetch/1.0.0/xfetch/
- https://medium.com/coinmonks/tao-facebooks-distributed-database-for-social-graph-c2b45f5346ea
- [缓存中的穿透、并发及雪崩](https://segmentfault.com/a/1190000013686915)
- [Golang’s Superior Cache Solution to Memcached and Redis](https://www.mailgun.com/blog/golangs-superior-cache-solution-memcached-redis/)
- http://lanlingzi.cn/post/technical/2018/0624_cache_design/
- https://github.com/doocs/advanced-java/blob/master/docs/high-concurrency/redis-caching-avalanche-and-caching-penetration.md
