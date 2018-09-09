title: 記一次 Elasticsearch troubleshooting 的歷程
date: 2018-08-15 22:57:44
tags: elasticsearch
---

## 過程

前陣子又發生了 AWS Elasticsearch 的 status 變成 red 的情況，這次跟以往的情形有點不一樣，之前爆炸都是因為 disk space 不足，而後來增加了 Curator 定期清理資料後就解了，而這次發生的 outage 有點不同，也讓我想要記錄下發生的原因和解法。

## 調查

在開了 Support ticket 和查詢 AWS 的 document 後，初步有了一些方向，根據文件上面寫的 A red cluster status means that at least one primary shard and its replicas are not allocated to a node，其實就是跟 shard 是不是運作正常有關係，這邊更新一些 Elasticsearch 的科普知識，Elasticsearch 的 document 其實是放在 index 裡面，而 index 會根據你設定的 shard 數量，把 document 平均分散到不同 shard 中，然後把不同的 shard 放在不同的 node 中，以求可以分散式的去請求資料，而在沒有調整過的 AWS elasticsearch 中預設的 shard 數量是 5，所以每次 create 一個新的 index 就會產生 5 個 shard，經過 AWS support 的調查，我們家的 elasticsearch 裡面共有 10654 shard (抖)，而太多的 shard 接著就造成 CPU utilization 越來越重，最後重到某個 node 存取不到後 (猜想該 node 應該是炸掉了)，某個 shard 又沒有即時產生好 replica 存在另外一個 node 上，就這樣 status red recovery 失敗。

## Troubleshooting 技巧

1. 使用 `GET /_cluster/allocation/explain` 可以看到 cluster 裡面哪些 assigned shard 出問題，還有原因是什麼
2. 使用 `GET /_cat/indices?v` 可以看到哪些 index 是有問題的

``` 
health status index            uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test1            30h1EiMvS5uAFr2t5CEVoQ   5   0        820            0       14mb           14mb
green  open   test2            sdIxs_WDT56afFGu5KPbFQ   1   0          0            0       233b           233b
green  open   test3            GGRZp_TBRZuSaZpAGk2pmw   1   1          2            0     14.7kb          7.3kb
red    open   test4            BJxfAErbTtu5HBjIXJV_7A   1   0
green  open   test5            _8C6MIXOSxCqVYicH3jsEA   1   0          7            0     24.3kb         24.3kb
```

3. `cannot allocate because a previous copy of the primary shard existed but can no longer be found on the nodes in the cluster` 基本上我們家遇到這個訊息在講的就是 primary shard 跟著 node 失蹤了，如果該 node 沒有回來，而你的資料很重要，可能就要從 snapshot 裡面撈回來

## 解決方法

我們基本上爛掉的 elasticsearch 是拿來存 application log 的，所以該 index 壞掉其實不太會影響線上資料，而按照 AWS support 的教學和查詢的一些文章，我們的解法如下

1. 砍掉該壞掉的 index ，`curl -XDELETE <cluster>/<index_name>`
2. 砍掉 older/unused/smaller 的 index 減少 cluster loading
3. 更改設定減少每個 index 的 shard 的數量

其他學習到的東西

1. 根據需求，可以使用 dedicated master node 增加系統的 stability
2. 越少的 shard 系統會越穩定，10 shards gives great performance, 100 gives good performance, 500 gives okay performance, 1000 gives bad performance and over 2000 is when the cluster begins to become unstable.
3. 還有讓 shard 的 size 保持在 10GB~50GB 的大小，會讓 query 的 performance 比較好
4. 以下的兩篇 reference 有講到很多其他 troubleshooting 的技巧

Reference
[https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/aes-handling-errors.html#aes-handling-errors-red-cluster-status](https://docs.aws.amazon.com/elasticsearch-service/latest/developerguide/aes-handling-errors.html#aes-handling-errors-red-cluster-status)
[https://www.elastic.co/blog/red-elasticsearch-cluster-panic-no-longer](https://www.elastic.co/blog/red-elasticsearch-cluster-panic-no-longer)
