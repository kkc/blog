title: MySQL partition 筆記
date: 2017-07-07 23:35:05
tags: mysql
---

最近要處理一件資料量不太大但是又超過 5kw 行的問題，問題是這樣的，需要一張 table 儲存上傳到 S3 影片的 metadata，像是 `id`, `timestamp`, `s3_url` 等等資料，然後過三個月就把之前的資料連同 metadata 砍掉，而想到簡單一點就用 mysql 的分區表來做就夠了，有人會問用 mongodb 分月切就好，但是其實在處理跨月的 range query 就需要自己在 application 端組合，相較之下 mysql 是在同一個 logic view 下面，對於程式邏輯可以變得比較簡單，更者可以直接用 ORM 來操作。

# 分區表 #

基本上我是用 AWS Aurora ，但主要還是參考了 [mysql 文件](https://dev.mysql.com/doc/refman/5.6/en/partitioning.html)，因為 Aurora 是用 mysql 5.6 模改的。
如同前言所講的，我需要處理一定量級的資料，但在資料量大到一定程度時，除非使用 covering index，要不然就會產生很多 disk 的 IO 去找資料，對於 performance 的下降是可以預期的，MySQL 的 Partitioning 會根據條件切分不同資料到不同的文件下，好處是在搜索時，index tree 變小了，搜索的資料量也變小了，插入時因為 index tree 沒那麼肥，也可以得到一定程度的改進，另外最大的好處就是移除分區表不會讓資料表造成碎片，很適合需要定期砍掉資料的場景。

## 建立分區表

可以使用下列 command 建立 table 與分區表

```
CREATE TABLE sales {
    ...
    created_at DATETIME NOT NULL
} ENGINE=InnoDB PARTITION BY RANGE(YEAR(created_at)) (
    PARTITION p2015 VALUES LESS THAN (2015)
    PARTITION p2016 VALUES LESS THAN (2016)
    PARTITION p2017 VALUES LESS THAN (2017)
    PARTITION p9999 VALUES LESS THAN MAXVALUE
)
```

這樣就可以對於 year 來切分 partition，然後我個人踩過幾個雷，基本上只能用 `YEAR` 和 `TO_DAYS` 來切，要產生根據月份切資料就要自己帶入 `TO_DAYS('2017-07-01')` 這種語法，然後用 `MONTH` 語法產生的 partition 基本上是完全沒用的，因為查詢時他還是會查全部的 partition ，也不能用 Prune 其中的 partition，[官方文件](https://dev.mysql.com/doc/refman/5.6/en/partitioning-pruning.html)也有寫到 Prune 的議題 `Pruning can also be applied for tables partitioned on a DATE or DATETIME column when the partitioning expression uses the YEAR() or TO_DAYS() function.`

## 分區表的搜索

要使用 `WHERE` 去定位使用哪個分區表，用以上的例子就是使用 `SELECT * FROM sales WHERE created_at >= '2017-06-01' AND created_at <= '2017-07-01';`

要看看使用了哪些分區，可以簡單用 `Explain partitions SELECT * FROM sales WHERE created_at >= '2017-06-01' AND created_at <= '2017-07-01';`

## 移除分區表

在維護 time-series 的 case 時，使用 `DROP PARTITION` 會比直接 `DELETE` 快上非常多！

## 插入分區表

如果 partition key 插入了 `NULL` 的值會導致過濾失效，然後全部資料都會掉入第一個分區表，所以要避免這個情況。
在 MySQL 5.5 後可以使用 `PARTITION BY RANGE COLUMNS(created_date)` 解決這個問題。

## Rotate Partition

在經過一段時間後，我們需要更動分區表，類似新增需要的分區表，還有刪除不必要的分區表，這時候可以使用 stored procedure + mysql event scheduler 來做，這裡有一個 stackoverflow 的[範例](https://dba.stackexchange.com/questions/90863/whats-the-best-way-to-structure-logging-application-data-in-mysql)，照著改還算簡單，只是 mysql 的 stored procedure 真的有點難 debug。

# Tips

有一篇文章叫做 [PARTITION Maintenance in MySQL](http://mysql.rjweb.org/doc.php/partitionmaint)，個人覺得還不錯，看了以後可以免去踩一些雷。
1. 除非資料量級到 > 1M rows ，在考慮使用 mysql partition
2. 不要開太多 partition (這個應該在 5.6 5.7 就不是太大的問題了)
3. 只有 PARTITION BY RANGE 有用 (雖然 partition 還有其他 key, hash, list 的方法，但是真的很不實用 hmmm)
4. SUBPARTITION 不好用 (同感，有點難維護)
5. Partition key 不應該是 index key 的第一個
6. Partition key 需要是 Primary key 或是 Unique Key
7. 如果 Partition key 是 dt ，通常 index 長得像 PRIMARY KEY (..., dt), UNIQUE KEY (..., dt)，會將他放在最後。

# Reference

1. [我必须得告诉大家的MySQL优化原理2](http://www.jianshu.com/p/01b9f028d9c7)
2. [PARTITION Maintenance in MySQL](http://mysql.rjweb.org/doc.php/partitionmaint)
3. [everything-you-need-to-know-about-mysql-partitions](http://www.vertabelo.com/blog/technical-articles/everything-you-need-to-know-about-mysql-partitions)

