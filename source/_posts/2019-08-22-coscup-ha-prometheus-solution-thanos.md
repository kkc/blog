---
title: Coscup 分享 - HA Prometheus Solution Thanos
catalog: true
date: 2019-08-22 10:23:55
subtitle: Thanos 真的很不錯！
header-img: ./star.jpeg
tags:
  - Thanos
  - Prometheus
  - CNCF
---

上個月參加了 Coscup，完成了我的 Coscup 講者處女秀，對比三年前當主持人，其實當講者輕鬆了不少，而且看到很多熟面孔的感覺非常好。

這次參加的 SDN x Cloud Native x Golang 議程軌，其實有非常多的好主題，而我也分享了一個跟 CNCF & Golang 有相關的 Opensource Project - Thanos，Thanos 主要就是為了解決 Prometheus 的 Global View & High Availability & Long-term storage 的問題，一直以來 Prometheus 作為 Cloud Native 主要監控元件，在經過社群的努力下，其實以單台 Prometheus 而言，效能和儲存效率上已經獲得很大的改善，目前是非常成熟的方案，但是在談到在部署多個 Prometheus 的情況時，往往會遇到一些問題像是，如何透過 PromQL Query 不同台的 Prometheus 並且 aggregate/merge 其中的資料，另外是 long-term storage 的問題，像是如何將歷史資料保存起來，而不是只有寫在 Prometheus 單體的 SSD 上面，這幾個問題就造就了 Thanos, Cortex, Uber M3 等等 Opensource 的存在。

## HA prometheus solution - Thanos

我的投影片分享如下
https://docs.google.com/presentation/d/1KBs4FxYwFL6dsz_JUbPK4ZiKXYjsaLZI21VgVLI54I4

前面幾頁就在講解單體 Prometheus 的問題，而就算使用了 Federate 之後，這個架構還是會有其他的問題，像是資料會被重複儲存在兩個地方，還有被拉取的 Prometheus 機器時也有可能發生 timeout 而很多情況下我們可能不會把所有的東西都拉到該機器上，另外壓力都會落在 Federate 起來的那台機器上面，這樣一來又還是需要到個別的 Prometheus 機器上面去做 Query，造成很多管理上面的不便。

而 Thanos 主要實現了三個願望
- Have a global view
- Have an HA in place
- Unlimited retention


### Global View & HA
![img](./ha.png)
![img](./ha2.png)

可以看這張圖比較下使用 Thanos 取代 Prometheus Federate，基本上 Thanos 使用 Sidecar Pattern，就算你有既有的 Prometheus 正在跑著，也可以透過 Sidecar 這個元件去做讀取，Querier 這個元件使用了 Thanos 定義的 StoreAPI 從 Sidecar 中讀取資料，Querier 裡面有 Deduplicate 和 Merge 的功能，所以也不用怕資料散在不同的 Promethues 上面，Deduplicate 主要是可以透過 Label 去認出相同的資料，這樣就不會重複把同一條線畫出來。

透過這種架構，可以很輕鬆的達成 HA，開兩台 Prometheus 去撈取資料，就算一台 Prometheus 掛掉，Thanos Querier 還是可以讀取其中一台，然後兩台都活著的時候， Deduplicate 可以將資料去重。

### Unlimited Retention
![img](./retention.png)
![img](./retention2.png)

Thanos 實現 Umlimited Retention 的方式也相當的簡單，就是利用 Sidecar 把 Prometheus 裡面的 Block 讀出來寫到 Object Storage，然後再提供一個 Store 的元件，用來讀取 Object Storage 裡面的 Blocks，這邊很聰明的地方就是 Querier 都是透過 StoreAPI 去做讀取，這個介面一致化後，其實讓 Thanos 變得相當有彈性。

這邊要寫一下要注意的地方，就是 Prometheus 其實預設是每兩個小時才會把 Memory 裡面的 Block 寫進 local storage，之後 Thanos Sidecar 才有機會將其上傳到 block storage，如果在中途你的 Prometheus crush 了，這樣就會有兩個小時的資料遺失了，所以 Thanos 官網上面其實是建議 Prometheus 如果跑在 k8s 上面，最好是掛著專屬的 PVC，這樣 Prometheus 回來的時候，還可以透過 WAL 回復資料。另外一個雷是 Prometheus 原本 Remote Read 沒有提供 Steaming 的格式，而 Sidecar 在讀取的時候，如果拉了一個 range 超大的資料，會造成 Prometheus OOM，而這個問題也在上個月的這個 [PR](https://github.com/prometheus/prometheus/commit/48b2c9c8eae2d4a286d8e9384c2918aefd41d8de) 解決惹。

## Other Components

基本上使用 Querier, Sidecar, 還有 Store 就可以完成很多的事情，但其實 Thanos 還有提供更多的功能，這邊我介紹了 Compactor 和 Ruler。

### Compactor
![img](./compactor.png)

Prometheus 的 TSDB 在改寫後，就有提供 Compaction (壓縮)的功能，基本上就是把 Memory 裡面的資料，透過 delta-of-data & XOR 的方式壓縮，這裡面參考了 Facebook 2015 年發表的論文 [<Gorilla: A Fast, Scalable, In-Memory Time Series DataBase>](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.718.197&rep=rep1&type=pdf)，有興趣的人可以看看，而透過這種壓縮方式，Prometheus 可以輕易地儲存很多的 series 以及保存很長的一段時間，而 Thanos compactor 乍看之下好像沒什麼用處，但其實它復用了 Prometheus 的 compactor library，並且在上面擴展了 Downsampling 的功能，也就是將這些 Blocks aggregate 成 5mins 和 1 hours，這樣的做法，可以讓讀取長時間的資料時，可以更快的取出資料，使用的 memory 也會變少，舉例來說，你想要看個半年的資料時，其實沒必要看到 raw data 那麼小的 resolution，其實只要透過 1hours 的資料就可以反推出趨勢，另外 Compactor 會幫忙管理資料的刪除，透過一個 Compactor 管理移除 Block Storage 的資料，其實也是比較好的做法。

不過在使用 Compactor 時，其實也有一個雷，就是要把 Prometheus 上面的 Compaction 關閉，要不然 Thanos 的 Compactor 還需要多做一步將資料還原才能做 Downsampling。

### Ruler
![img](./ruler.png)

Ruler 這個元件其實是為了擴展 Alertmanager 而用的，因為使用 Thanos 後，在設定 Prometheus 上，可能會把超過 2 小時的 Block 儲存到 Block Storage 上，然後把 Prometheus 自身的 Retention 關小，如此一來你在 Prometheus 上面設定的 Alert rule 如果觀察的趨勢是超過 2 小時的，就很有可能會失效，另外是在 Query 不同 Cluster 上面，就沒辦法設定一個 Rule 去覆蓋所有的 Cluster，Ruler 給了我們這樣的彈性，可以將 Alert rules 都集中給 Ruler 管理。

## Conclusion

在投影片中，我還有列舉了一些官網上面建議的部署模式，可以看到 Thanos 也支援一些複雜的情境，然後其實已經有蠻多大公司都用在 Production 上面，所以算是一個成熟的方案，蠻推薦大家可以玩玩看。

Thanos 在七月的時候，也被捐出來給 CNCF，正式成為 CNCF Sandbox 的專案，有了更多資源後，我們可以預期他會越來越好用，社群的人解 issues 和 feedback 的速度也很快，有心玩 golang open source 的人，我覺得 Thanos 是蠻好的一個專案。