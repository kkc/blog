title: Mongodb 如何使用Memory和tuning的小技巧
date: 2015-07-21 13:34:36
tags: mongodb
---

開始翻以前在 evernote 的連結，發現這篇對於 mongodb 如何使用 memory 的介紹算是簡單易懂，這邊做個小筆記

<iframe src="//www.slideshare.net/slideshow/embed_code/key/J7LaXFlwQkAgM" width="425" height="355" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/mongodb/mongodb-london-2013understanding-mongodb-storage-for-performance-and-data-safety-by-christian-kvalheim-10gen" title="MongoDB London 2013:Understanding MongoDB Storage for Performance and Data Safety by Christian Kvalheim, 10gen " target="_blank">MongoDB London 2013:Understanding MongoDB Storage for Performance and Data Safety by Christian Kvalheim, 10gen </a> </strong> from <strong><a href="//www.slideshare.net/mongodb" target="_blank">MongoDB</a></strong> </div>

## mmap ##

Mongodb 的設計是利用`mmap`方式映射檔案文件到 memory 中，直接透過操作 memory 來存取資料，好處有下列幾點:

1. 簡化設計，省略最複雜的 memory 和硬碟管理，讓 Mongodb 的開發變得容易許多。
2. OS 可以對不同文件系統(ext3, ext4)的類型做 cache。
3. 原生的 OS LRU cache 機制。
4. Mongodb 重開後可以繼續使用 file cache 裡面的資料。

而缺點也很明顯:

1. 因為檔案系統和 memory 是一對一映射，就如前篇所說的，檔案系統內的 fragment 會一同被複製到 memory 中, 造成 memory 的浪費。
2. 太大的 Linux `readahead` 會對 memory 的使用造成影響。readahead 指的是一種 prefetching 技術，在檔案讀取中，因為 data 會有 spatial locality (空間局部性)，所以當我們讀取資料時，會一次讀一大段連續資料進來，類似一次讀 16k 或 32k 進入page cache，去減少硬碟速度過慢造成的影響，然後增加檔案的速度。
3. LRU 的 cache 方式，不能夠去設定 Priority，像是 index 這類資料，也是會被 LRU 方式被 swap out。

## Resident memory ##

通常我們會想知道 data 到底使用了多少 memory，而比較好的指標是 Resident memory，基於 Mongodb 是用 share memory 的機制， 查看每個 Mongodb 的 process 的 Resident memory 都會是一樣的。

Resident memory 的組合如下

{% blockquote %}
Resident memory = process overhead + File system pages 被 access 的部分
process overhead = connection + heap
{% endblockquote %}

有些人去檢查後會覺得很奇怪，為什麼 Resident memory 相對於全部 memory 而言少那麼多，以為 Mongodb 沒有完整的使用 memory， 其實不用擔心，mongodb 會把資料都讀入 file cache ，這點可以用`free -m`來檢查，是否`cached`那欄吃了不少 memory ， 而 resident 裡面會顯示的空間，則是 mongodb 真正有 access 到的檔案，這部分的數據會被 memory fragment 和 readahead 影響, 如果發現 Resident memory 過低, 應該是要想辦法減少 memory fragment 或是調整 readahead 的大小， 讓讀進來的資料都是需要的, 減少發生 page fault 的機率。

## Working set & page fault ##

常聽人說 Mongodb 需要大量的 memory， 最好是 memory 要能夠 fit working set，但大家常以為 working set 就代表全部資料的大小，這是個錯誤觀念。實際上 working set 指得是 mongodb 完成所有操作要取用的 data 和 index 大小。例如資料庫內不是每筆資料都常常被 access，舉 Blog 來說三年前的資料就不會那麼常被存取, 那 working set 裡面就不需要包含那麼古老的資料。

查看 working set 的指令`db.runCommand( { serverStatus: 1, workingSet: 1 } )`

在 mongodb 中， working set 的資料如果不在 memory 中，就會引發 page fault，這個時候就要去觀察 page fault 的大小， page fault 如果超過一定值(ex. 200以上)， 然後 latency 又受到影響， 你的系統可能就需要一些調整了，有可能是要加RAM，也有可能是資料沒有上到 index 而造成 Table scan， 而下一節我也有列出一些平常我會調整的部分。

## Tunning ##

以下是我在AWS EC2平台上的一些調整方法

### 記憶體 ###

1. 設定swap（預設為60, 當系統使用到超過40% memory, 就會嘗試使用swap）
    ```
    sysctl -w vm.swappiness=1   # from 60 -> 1
    ```

2. Huge Page (especially THP) for mongodb: https://jira.mongodb.org/browse/DOCS-2131
    ```
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
    ```

### File System ###

1. File system flush issues (有興趣可以查看[這篇](http://blog.littlero.se/post/linux-tuning-for-write-heavy-system/))
    ```
    # vm.dirty_ratio = 80                  # from 40
    # vm.dirty_background_ratio = 5        # from 10
    # vm.dirty_expire_centisecs = 12000    # from 3000
    ```
2. For Mongodb Disk Mounting

    modify `/etc/fstab`
    把journal和一般的data掛在不同的disk上面, 因為我是用SSD硬碟, 所以取消atime, 然後加上discard的指令, 讓SSD開始Trim的功能

    ```
    /dev/xvdf /mongodb_data ext4 defaults,auto,discard,noatime,noexec 0 0
    /dev/xvdg /journal ext4 defaults,auto,discard,noatime,noexec 0 0
    /dev/xvdh /log ext4 defaults,auto,discard,noatime,noexec 0 0
    ```

3. 設定 Readahead

    Readahead 根據資料不同, 需要有不同的調整, mongodb官網是建議在32k以下, 而最後不要低於16k, 因為Mongodb的index bucket的大小就是8k, 調整太低可能會失去一些好處。

    ```
    sudo blockdev --setra 32 /dev/xvdf
    ```

4. ulimit

    修改`/etc/security/limits.conf`

    ```
    * soft nofile 64000
    * hard nofile 64000
    * soft nproc 32000
    * hard nproc 32000
    ```

5. IO scheduler 因為是使用SSD based的硬碟, 採用Noop

    ```
    echo noop | sudo tee /sys/block/xvdf/queue/scheduler
    ```
