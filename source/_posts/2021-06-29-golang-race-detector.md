---
title: Golang Taipei Gathering 分享 - Golang Race Detector under the hood
catalog: true
comments: true
indexing: true
header-img: ../../../../img/default.jpg
top: false
tocnum: true
date: 2021-06-29 21:59:17
subtitle:
tags:
  - golang
categories:
---

## Preface

這個月初在 [Golang Taipei Gathering](https://www.meetup.com/golang-taipei-meetup/events/278400349/) 分享了最近的讀書心得，Go race detector 底層到底是怎麼實作的，其實主要是有在追一個大神 [Dmitry Vyukov](https://twitter.com/dvyukov) 的推特，而他一直以來都貢獻了很多 dynamic tools 像是 AddressSanitizer，MemorySanitizer 和 ThreadSanitizer，另外還有一些 run fuzzy tests 的工具，都可以用來找不好找的程式錯誤。基本上 go race detector 也是由 Dmitry 發起，建構在他自己維護的 C++ ThreadSanitizer 上面。另外一點是從 race detector 的文件上面，知道 race detector 不會有 false positive，但是有可能有 false negative (該找到的 race 沒找到)，於是就想研究看看怎麼運作的，研究後覺得很有趣，順手整理整理分享給大家。

## 投影片 & talk

- Slides: [Go Race Detector Under The Hood](https://speakerdeck.com/kkc/go-race-detector-under-the-hood)
- [Talk](https://www.youtube.com/watch?v=Qs4yuL-c0H0&t=1550s)

其實很多內容是從下列兩個 talk 來的，其中的範例有問過 kayva 的同意使用：
https://www.slideshare.net/InfoQ/looking-inside-a-race-detector
https://krakensystems.co/blog/2019/golang-race-detection

### Data race vs Race condition

一開始我是先用這個主題破題，因為很多人其實會把 data race 和 race condition 搞混，data race 是有具體定義的

> When 2+ threads concurrently access a shared memory location, at least one access is write.

race condition 則是指更高階的行為，當不同的 threads 在一系列的 operations 後，因為操作的順序(order)不同，導致 program behavior 或是產生的 result 不符合預期。

### 一個 Race condition 的範例 

從這個[文章](https://blog.regehr.org/archives/490)可以找到一個有趣的範例，裡面的程式沒有 Data race，但是有 Race condition:

```
transfer2 (amount, account_from, account_to) {
  atomic {
    bal = account_from.balance;
  }
  if (bal < amount) return NOPE;
  atomic {
    account_to.balance += amount;
  }
  atomic {
    account_from.balance -= amount;
  }
  return YEP;
}
```

可以看到裡面每個 variable 都有用 atomic 去保護，所以不會有多個 threads 讀寫同一個 memory location 的情況，但是在多個 threads 運作下，這個轉帳程式其實是錯的。

## Happens-before

要了解 Race detector 怎麼運作需要先理解 Golang 的 [memory model](https://golang.org/ref/mem)，
尤其是所謂的 happens-before 的關係，一般來說，golang 在單個 goroutine 的情況下比較單純，因為有 ILP (instruction level parallelism) 的情況下，指令是有機會被重排的，這邊只需要保證程式最後的輸出行為不變就好，而在 multiple goroutine 的情況下，則需要透過 built-in 的 synchronization primitives 去建立 happens-before 的關係，這樣可以保證 shared variable 沒有同時有 read/write access 的情況，race detector 也是藉由去 trace 這樣的關係，找到可能的 data race 的 variables。

## Race detection algorithm & vector clock

其實 Race detection algorithm 一直是個學術問題，過去那麼多年也是蠻多人在研究的，實際上已經被證明是個 [NP-Hard](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.40.9765) 問題，其中又分為 static & dynamic 的分析，static 的分析基本上很花時間，而 golang 的 race dector 採取的是 dynamic 的方法，也就是在程式執行過程中，經由不斷地紀錄 variable 的狀態，去看看有沒有 data race。

我在 talk 中有分享下其底層 ThreadSanitizer 其實是由 [DJIT+](https://dl.acm.org/doi/10.1145/781498.781529) & [FastTrack](https://dl.acm.org/doi/10.1145/1543135.1542490) 演變而來，這兩個演算法也是蠻久以前就被提出的 (2003 & 2009)，而最值得一提的部分，則是其精神是用分散式系統大神 Lamport 的 [vector clock](http://lamport.azurewebsites.net/pubs/time-clocks.pdf) 去紀錄每個 variable 被 thread access 的時間，並且透過 synchronization primitives 讓 variable 去 sync 不同 thread 間的 vector clock，再從中去分析是否有 data race。

這部分其實乍看下蠻複雜的，不過其實論文裡面給的證明和例子蠻清楚的，很推薦大家直接去看，反倒是我用寫的還寫不出來，詳情可以參考我的 talk & slides ，裡面有逐步的分解，希望有把這部分講清楚。另外是我在投影片中為了使用 kavya 的例子，所以用了跟論文不同的符號，這方面我還需要再改進改進。

### 額外補充
- [L.Lamport Time, clocks, and the ordering of events in a distributed system. 1978](https://lamport.azurewebsites.net/pubs/time-clocks.pdf)
- [Read a paper: Time, clocks, and the ordering of events in a distributed system](https://www.youtube.com/watch?v=ummQWQgkWOA)

## 簡單的案例

在 talk 中我分析了下面這個 case，讓大家想想這個 case ，當 goroutine1 和 goroutine2 同時執行時，會不會有 data race 的情況：

```golang
var count = 0
var mu sync.Mutex 

// go routine1
func incrementCount1() {
  mu.Lock()
  count += 1
  mu.UnLock()
}

// go routine2
func incrementCount2() {
  mu.Lock()
  mu.UnLock()
  count += 1
}
```

而其實答案是 it deponds，根據 race detector 的走法，如果先走 go routine1 在走 goroutine2，這個 data race 檢查不出來，因為建立完 happens-before 後，count 這個變數並沒有被 concurrent access，而先走 goroutine2 在走 goroutine1 則是會被 race detector 抓出來，因為演算法檢查 vector clock 就可以知道他們有 concurrent access 的關係。

而這個情況下，我們就知道 golang 實作的 race detector 為什麼會有 false negative 的情況，而這類的 case 可能就是多跑幾次，在不同的組合下才有可能抓到，但是優點是比其他的 race detection 演算法還要快很多，而且 memory 用量也比較少。

## ThreadSanitizer

ThreadSanitizer 其實是 C++ library，而透過 compiler instrumentation 的技術，可以將 golang 程式在讀寫變數的地方，插入 instrumentation code 去 invoke ThreadSanitizer 的 functions，主要紀錄狀態和 state machine 的地方都實做在 ThreadSanitizer 裡面了。

另外是透過 shadow state 的技術，去紀錄 variable 被 access 的狀態，這也是一個經典的地方，這類的技巧廣泛地使用在 valgrind, memory sanitizer 和 address sanitizer，而每個 8 bytes 的 memory location，都額外再使用了 4 個 shadow values 去紀錄狀態， 所以低消就是會比你的程式多吃 4 倍的記憶體，

## 心得

第一次線上給 talk 其實感覺蠻奇怪的，好像自言自語，在看不到觀眾情況下，還是不太能拿捏節奏，一般來說我喜歡設計一些小問題，讓觀眾能夠想一下，再藉由觀眾的表情了解我到底有沒有講清楚，尤其中間講到一半還有越來越虛的問題，一直很怕是不是網路不穩大家都沒回應。不過這次的 talk 是利用 google meet 進行，在 Evan 大好心的貼到 twitter 上面分享，希望有更多人可以進來聽的時候，居然混進來一些外國人來亂，原本還擔心我講的一半有人會睡著，沒想到居然發生這個插曲，也是蠻好笑的。


## References

- https://krakensystems.co/blog/2019/golang-race-detection
- https://www.slideshare.net/InfoQ/looking-inside-a-race-detector
- https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm
- https://stackoverflow.com/questions/61674317/what-are-shadow-bytes-in-addresssanitizer-and-how-should-i-interpret-them
