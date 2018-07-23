title: Mongodb Storage 的架構和問題解法
date: 2015-07-14 12:05:29
tags: mongodb
---

Mongodb 的 Storage 架構其實很簡單，這篇文章其實是看完 mongolab [這篇](http://blog.mongolab.com/2014/01/how-big-is-your-mongodb/)和自己一些經驗心得。

## MongoDB storage 架構 ##

### Database and Extents ###

首先參考下面這張圖(與原文不同，我稍微做了點修改)

![mongodb_storage](/img/mongodb_storage.png)

假設我們有個 database 叫做 test，裡面有兩個 collection (A & B)，而基本上會被這樣儲存，test.0 & test.1 這些 data files 就是真正佔據硬碟的空間，mongodb 會先 preallocate 這些空間，主要有兩個好處：

1. 減少 disk 空間分配的 fragmentation
2. 減少寫入時，再去跟硬碟要空間分配的動作

缺點是不知道一開始要 allocate 多少空間才是最完美的，allocate 太少就少了第二點好處，allocate 太多沒用完又會浪費硬碟空間。 mongodb 預設採用的方式是 db.0 為 64mb，db.1 為 128mb，db.2 為 256mb，一直到了db.5 才會變成2G， 而這些根據需求都可以另外設定。

每個 data file 又有很多的 extent，index 和 data extents 會分開存放，然後存不下的時候，才會被放到新的data file。

### 觀察 Database 大小 ###

使用mongodb的人一定要學會用 db.stats() 去觀察資料大小，而 db.stats() 又包含了三個觀測指標 dataSize、storageSize 和 fileSize。

#### DataSize ####

所謂datasize，指的是上面橘色的部分，其中包含了全部的 documents + padding 的空間。

當砍掉 document 的時候，datasize 的值會下降，而改變 document 的內容（包含減少key）則不會改變 datasize 的大小。另外要注意的是，增加 document 的大小時，如果超過 padding (預先分配) 的大小， 會重新找一塊比較大的區域插入，因為 document 通常是連續插入的，如果 document 的大小不一， 原本刪除的地方，有很大的機率會留下洞，也就是新的 document 就無法插入，該區域就浪費了。

這邊想要再補充一下，因為 mongodb 是利用 mmap 的方式讀取資料進入 memory ，而這些 document 的洞，也會變成 memory 上面的洞 (fragmentation)， 所以我們最好要避免這些洞，進而去減少 page fault 發生的機率。

#### StorageSize ####

Storage Size 指的是上圖藍色的部分，也就是 database 中，所有預先分配給 document 儲存的地方。 所以在刪除 document 時，Storage size 並不會減少， 而當增加 document 時，如果原本欲分配的空間不足時， 其大小才有可能會增加。

#### FileSize ####

File Size 就更簡單了，就是整個 database 佔據的大小，包含了data & index 的空間，而 FileSize 只有在砍掉 database 時才會減少，刪除 collection、document、index 對他都不會有任何的影響。

## 問題 ##

### collection 裡面的fragmentation ###

在講解 Datasize 的時候有提到，砍掉舊的 document 時，要是新的 document 太肥塞不進去，就會造成 fragmentation ， 甚至會變成 memory 上面的 fragmentation ，進而影響 mongodb 的效能，一般我們會去看 DataSize 和 StorageSize 是不是差距過大去 確認是否有這個問題。

要解決這樣的問題，其中一個方法是使用compact，但是這個指令會 block 整台機器。另外一個方法就是插入 document 時，都是用同樣大小的 size，這樣刪除 document 後才有機會被 reuse。


### 清 fragmentation 的其他方法 ###

1. 除了 compact 和 db.repairDatabase 外，還有個可以清除 fragmentation 的方式，其實是建立新的 replica set node ， 因為在建立 replica set 的時候，資料會重新寫入，所以資料空間可以有比較好的利用率，減少不必要的 fragmentation ， 不過使用這招也是有點麻煩，因為在建立 replica 的時候， mongodb會吃掉大半的 Disk IOPS, 反應會變得比較慢， 而且 mongodb 的資料都沒有壓縮過，所以要 sync 全部的資料庫，需要非常久的時間。

2. 將 secondary replica node 下線後, 先拿來清 fragmentation ，接著再讓他上線sync完資料後，切換成master。

3. Migrate 舊的 database 成為新的 database，優點是不需要人工操作太多 mongodb replica 的工，但缺點是 database migration 需要考慮會不會影響線上的服務，如果是24/7的服務，程式碼要就要同時讀取舊的資料庫和新的資料庫，確保資料同步。

## 結語 ##

很多使用 mongodb 的人不知道自己的 db fragmentation 情況很嚴重，往往機器變慢了也不知道原因，最好的方式是要定期檢查上面 三個指標，注意 DataSize 遠小於 StorageSize，或是 StorageSize + index Size 遠小於 FileSize的情況，然後盡量把document的大小固定， 相信可以有效地改善這些問題。
