---
title: google 的 swisstable hashmap 筆記
catalog: true
date: 2020-10-17 12:25:14
subtitle:
header-img:
tags:
  - swisstable
  - hashmap
---

Matt Kulukundis 在 cppcon 2017 年給的 [talk](https://www.youtube.com/watch?v=ncHmEUmJZf4)，我覺得這個 talk 講得非常好，主要是說明如何使用 Swiss Tables ，設計出更符合當代硬體架構的 hash map (flat hash map)，google 內部大量採用這個改寫 std:unordered_map，我看完這個之後有點觀念被刷，覺得還蠻震驚的。

這個 hash map 的實作考量到了 cache-friendly 以及透過 SIMD 來加速，主要讓我學習到的是，以往很多 hash map 實作使用 chaining 是因為在 load factor 很高的情況下，可以減少 key collision 改善插入的時間，而 flat hash map 則是告訴我們使用 open addressing，將 element 都放進一個 flat memory array 裡面，其實是 cache-friendly，另外是透過 SIMD 可以一步就知道結果，而不用在 透過好幾次的 cpu cycle 找資料，接著就可以在搜索上面得到巨大的加速。

看到幾篇文章 [Swisstable, a Quick and Dirty Description](https://gankra.github.io/blah/hashbrown-tldr/) 把概念也講得蠻清楚的，也成功 port 這個 table 到 [Rust](https://github.com/rust-lang/hashbrown) 上面，稍微想了一下像這類的方法，Go 的架構因為要考慮 Runtime 還有沒有 Generics，似乎就比較難實作這塊，這邊有錯還請指正，

總的來說 google 開源的 Abseil [1] 和 Rust stdlib [2] 都有採用這個的資料結構，不過還是要注意一下自己的資料長怎麼樣，還有 key 通過 hash function 後的 distribution，來找到適合自己使用的 hash  map。

另外我現在才學習到有非常多 SIMD 的 algo，簡直是開了我的眼界，感覺還有非常多的東西可以使用這個加速，就讓我們再繼續看下去。

[1] https://github.com/abseil/abseil-cpp
[2] https://github.com/rust-lang/hashbrown
[3] https://rcoh.me/posts/hash-map-analysis/
[4] https://www.youtube.com/watch?v=ncHmEUmJZf4
