title: RDBMS 先修課程筆記
date: 2015-08-16 14:46:34
tags:
 - mysql
 - mongodb
---

這個禮拜一放棄DevOps聚會, 跑去上[TritonHo](https://www.facebook.com/ho.s.fung?fref=ufi)的 RDBMS 先修課程, 感覺收穫頗多的, 可惜沒時間在平日的時候繼續上, 把不足的知識補起來, 其實類似的課程還蠻推薦大家參加的, 因為很少有大大會出來無私分享。

以下是那天的投影片:
<iframe src="https://drive.google.com/file/d/0B-UTE7EObr6ydDZOZXNLRko5UTQ/preview" width="640" height="480"></iframe>

主要在談論 RDBMS 的一些基本概念, 還有這是一場破除 RDBMS 就是比 NoSQL 慢的演講, 講者談到幾個重點, 我個人覺得頗重要的:

1. NoSQL 常常是犧牲一些東西來換取速度, 類似安全性和正確性
2. 如果你的數據沒有很大, 可以不要使用 NoSQL, 因為使用 NoSQL 的公司如 FB, twitter, 資料量都是 PB 計算的(主要是 data migration 的問題)
3. 沒有 Transation 很多事情會很難辦

然後以下是我的筆記, 但感覺還是有點照抄投影片, 之後還想要補齊 NoSQL 做 Trasaction 的方法。

## 為何選擇RDBMS ##

1. 歷史悠久, 將近有30年歷史
    a. 社群支援
    b. 軟體錯誤較少
    c. 有許多用在 production 的經驗
2. 使用 RDBMS 比 NoSQL 開發速度快 (這點我存疑, 因為一般人就受了NoSQL的快速開發好處XD）
3. 中小型系統, 使用 RDBMS 效能就夠了(同意)
4. 使用 RDBMS 更安全

## 大部份系統需要atomic, multi-record, inter-dependent的操作 ##

像是轉帳問題: A扣掉十元, B加上十元

[這裏](http://stackoverflow.com/questions/10752993/how-do-you-do-atomic-multi-record-inter-dependent-operations-in-nosql)找到有人在討論 Mongodb 怎麼做這件事情, 真是有夠麻煩啊（翻桌）

## 特別有談到 RDBMS 支援十進制的 numeric, 計算金錢數值時相當有用 ##

MySQL 是有 decimal 這個屬性, 在查詢或是運算都不會有誤差, 寫程式的時候其實蠻害怕遇到浮點數的, 所以這點算是幫助頗大。

[does-mongodb-support-floating-point-types](http://stackoverflow.com/questions/7682714/does-mongodb-support-floating-point-types)
參考樓上這篇文章, Mongodb 有支援int32 (4bytes), int64 (8 bytes), double (64-bit IEEE 754 floating point), 而一般可以把數值存在double中, 順便提一下在mongodb的shell中, 因為是使用Javascript引擎, 好像分辨不出來 int 和 double ...Orz

## 如果數據規模小於100GB, 使用RDBMS當作報表系統就夠了 ##

1. Join & sub query 可以滿足大部份需求
2. 內建函數可以多加利用 (AVG, SUM, COUNT)

## 資料安全性 ##
RDBMS 所有返回 “成功” 的數據都確實寫到硬碟裡面了, 很多 NoSQL 的預設是只有寫到memory, 講者特別要提Mongodb... (不要再打臉啦, 腫腫的)

> Mongodb需要在 application 的 driver 中調整write concern, 還有要開啟journaling, 但開啟後寫入效能會有5%~30%的下降

## ACID ##
### Atomicity ###
- RDBMS 的操作都是以 Transaction (Tx) 為單位
- 一個 Tx 可以包含很多SQL指令
- 同一個 Tx 的訊息, 要就是照順序改動完成, 要不然同一 Tx 之前的操作都要 rollback
   > from wiki: Atomicity requires that each transaction be "all or nothing"
- 當機時, 還沒有被 committed 的 Tx 全部會被 Rollback ( 這邊感覺跟consistency比較有關）
- 當機後, 會依照 Tx 為單位, 重新復原資料庫
- 不要以為系統不會當機
- 在 Tx 中可以簡單做到帳戶轉帳 (只要簡單四行)
  ```
   START TRANSACTION;
   Update user_balance set balance = balance – amount where username = 'UserA';
   Update user_balance set balance = balance + amount where username = 'UserB';
   COMMIT;
  ```
- NoSQL 沒有transaction, 需要 implement 2 phase commit (要9個step, 投影片裡面有)

### Consistency ###
- RDBMS 內有unique constraint 和 用戶定義的constraint, 在 Tx commit時, 所有的 constraint都要符合 (這邊要再查查有那些constraint)
- 不符合constraint, RDBMS 會自動復原目前 Tx 的一切改動
- 使用RDBMS, 發生錯誤 (包含網路或主機問題) 而中斷一個Tx, 只要無腦把 Tx rollback 到一致性的狀態而 NoSQL 採用2PC, 在當機後要復原資料會很麻煩

### Isolation ###
- 同一筆資料, RDBMS 保障不會被兩個 Tx 同時改動
- Tx 可以同時進行, 但是每個 Tx 過程中要能看到一致的數據
- 不同的 Isolation level, 能讓 application 看到應該看的資料（？）
- 要避免 race condition, 上鎖 (LOCK) 是不能避免的, RDBMS 能自動處理
- 沒有 isolation 的機制的話, 要自己手動管理 locking
  >這裏舉了一個例子, 用同個帳號在同個時間下去領同個帳戶內的錢,  如果帳戶沒有上鎖, 錢可能被領成負的。
- 而在 RDBMS 中, 使用 update 會自動對受影響的數據加上write lock, 並且自動在 Tx 結束後釋放, 保證一份資料不會被同時改動
- RDBMS 也有支持 atomic check-and-set 模式, 所以檢查用戶是否有足夠錢和改動餘額的動作是Atomicity的
    ```
    update user set balance = balance - $amount where user_id = $user_id and balance >= $amount
    ```
- RDBMS 的 isolation 有包含全自動化的 lock management, 也有支持自動化的deadlock detection

### Duration ###
- 一旦 committed 的資料改動, 除非儲存空間受損, 否則不會流失
- 資料寫入時斷電, 如果已經寫入redo log內了, 在斷電後也可以復原資料
- MongoDB 的標準設定是沒有等待 Write 真正寫入成功
- 在 RDBMS 裡面, 系統斷電後, 會利用 redo log 來做 data recovery, 經過那麼多年的producton測試, 應該比較有保障

## ACID 我的小心得 ##

- Atomicity 保證系統能安全地移動到下個正確狀態
- Consistency 確保移動失敗時, 能夠正確的回復到原本的狀態
- Isolation 更改資料會自動上鎖, 不會發生杯具
- Duration 保證寫入成功的資料不會掉 (除非硬碟爆炸)
- RDBMS 能夠保證資料完整性

