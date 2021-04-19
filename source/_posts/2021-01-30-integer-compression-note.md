---
title: Integer Encoding Algorithm 筆記
catalog: true
date: 2021-01-30 22:13:14
subtitle:
header-img: compress.jpeg
tags:
  - algorithm
---

## Integer Encoding Algorithm 筆記

現今的電腦在 CPU 和 Memory 的速度上有極大的差距，而 Memory 到 Disk 上面的差距就更大了，所以有許多的壓縮演算法被套用在不同的應用上，例如 IoT, big data 和 database 之類的, 一個是為了節省儲存資料的空間，還有是小批的資料要 load 進 Memory 裡面處理也會比較快，這篇筆記探討的是像是 integer 類型的數據結構，有什麼樣的演算法可以套用，還是其中的差距是什麼，說到底要解的問題就是，就是給你一串連續的數字，要怎麼樣在壓縮率，encode/decode 的速度上面做改善。

這篇文章算是自我做的一個筆記，如果有寫錯還請指正，另外要指出以下的演算法都是 lossless data compression。

## History

這邊條列一下歷史，另外是我覺得蠻值得筆記的幾個演算法，主要從這篇論文來的 https://arxiv.org/pdf/1908.10598.pdf

1972: variable-byte (https://en.wikipedia.org/wiki/Variable-length_quantity)
1998: frame of reference (FOR)
2006: PForDelta
2009: Variant-GB
2010: simple8b
2011: Variant-G8IU
2014: Roaring
2015: SIMD-BP128, SIMD-FastPFor
2018: Stream-VByte

## 一些常見的 Algorithm

基本上討論壓縮率應該要從 bit-aligned, bytes-aligned, word-aligned 提起，不過 bit-aligned 演算法例如 Golomb coding or rice coding，雖然壓縮率很好，但是在 encode/decode 上面因為不符合電腦運作的模式[註1]，壓縮和解壓縮的速度很差，所以運用在資料庫上面的效果並不好。

註1: [你所不知道的 C 語言：記憶體管理、對齊及硬體特性](https://hackmd.io/@sysprog/c-memory)

>電腦的 cpu 又是如何抓取資料呢？cpu 不會一次只抓取 1 byte 的資料，因為這樣太慢了，如果有個資料型態是 int 的 資料，如果你只抓取 1 byte ,就必須要抓 4 次(int 為 4 byte)，有夠慢。所以 cpu 通常一次會取 4 byte(要看電腦的規格 32 位元的 cpu 一次可以讀取 32 bit 的資料，64 位元一次可以讀取 64 bit)，並且是按照順序取的


### Variable Byte

byte-aligend 顧名思義就是使用最少的 byte 儲存數字，類似原本一個數字是 4 bytes (32bit)，如果可以放進一個 byte 裡面，就可以省下很多的空間，VB 的運作模式很單純，利用 7 個 bits 存放資料，最前面的 1 個 bit 拿來判斷後面是不是有跟著其他 byte，以 128 的例子來看，最後一個 byte 的 binary format 是 10000000，但是我們只有 7bit 能拿來存資料，所以就需要兩個 bytes 把 128 存起來，第一個 byte 的開頭設定為 1，表示這個 byte 後面還有跟著另外一個 byte，到時候要一起拿來 decode 成原始的 binary。


|數值 | binary (32bit)  | Variant   |
|-----|----------|-----------|
| 0   | 00000000 00000000 00000000 00000000 | 00000000  |
| 1   | 00000000 00000000 00000000 00000001 | 00000001  |
| 127 | 00000000 00000000 00000000 01111111 | 01111111  |
| 128 | 00000000 00000000 00000000 10000000 | 10000001 00000000  |
|16383| 00000000 00000000 00111111 11111111 | 11111111 01111111|

以上的例子都是從 [wiki](https://en.wikipedia.org/wiki/Variable-length_quantity) 來的。

而 VB 作為一個很廣泛使用，又很好實做的演算法，也是有許多的改進的版本，例如 google 的 Jeff Dean 的 Challenges in Building Large-Scale Information Retrieval Systems [投影片](https://static.googleusercontent.com/media/research.google.com/en//people/jeff/WSDM09-keynote.pdf)裡面可以看到 Group Varint Encoding，藉由把 control bit 提到第一個 Byte 的前四個 bit，可以有效地減少 branch prediction miss 的 penalty，以提升 encode/decode 的速度。

而 2018 年又有新改良的版本叫做 stream VB (尚未研究)

參考資料: https://en.wikipedia.org/wiki/Variable-length_quantity

### Delta of Delta + Variable length coding

delta 也是一個很常見的技巧，像是 influx db 或是 prometheus 這類的 timeseries database 在壓縮 timestamp 時，也是會使用 delta 的方式將數值變小，例如說 [100, 101, 105, 108] 就可以轉成 [100, 1, 5, 8] 的格式，還可以進一步地把 [1, 5, 8] 變成 [1, 4, 3] 這種 `delta of delta` (DoD) 的格式。 

在 [Facebook Gorilla paper](https://www.vldb.org/pvldb/vol8/p1816-teller.pdf) 裡面也有提到，他們的 timestamp 也是用 DoD 壓縮出來的，主要他們的格式是 (DoD of time, value), (DoD of time, value) ... 聯合起來的，產生出來的 DoD 需要使用 Variable length coding (VLC) 的方式產生 tag bits 才能在 decode 的時候，知道那個 DoD 到底是幾個 bit ，像下面這張表:

| DoD     | tag bits | following bits   |
|---------|-----------|-----------------|
| 0       | 0         | 0               |
| [-63, 64] | 10  | 7 bits           |
| [-512,511] | 110 | 10 bits  |
| [-4096,4095] | 1110 | 13 bits  |
| [-32768,32767] | 11110 | 16 bits  |
| else | 11111 | 64 bits  |

所以在時間那個欄位會長成像下面這樣，透過 tag bits 可以跟 value 的值一起 encode 在一起
```
|time   value |   time    value  |     time      value |
| 100     1   |  '10': 2    1    |   '110': 100    10  |
```

### Run Length encoding (RLE)

[RLE](https://en.wikipedia.org/wiki/Run-length_encoding) 也是一種常見且簡單的壓縮格式，這邊的內容擷取自這篇[文章](https://wiki.multimedia.cx/index.php/Run_Length_Encoding)，

```
5 5 5 5 8 8 8 2 2 2 2 2
```
可以被 encode 為
```
4 5 3 8 5 2
```

這樣有了 50% 的壓縮率。 但是有個特例像是

```
1 2 3 4 5 6
```
如果被壓縮為
```
1 1 1 2 1 3 1 4 1 5 1 6
```

這樣反而比原本的資料慘，這時有個方法是可以使用下面的方法

```
-6 1 2 3 4 5 6
```

給他一個 indicator 讓他看到 -6 就知道要 copy 後面六個數字出來


### Simple family

simple family 也是一個很有名的演算法，因為名字的開頭就叫做 simple，而依據出現的是 simple9, simple16, 最後是 simple8b，simple8b 的 paper 出現在 2010，目前被廣泛使用在 timeseries database，也算是一個相對新的演算法，核心思想是把數字壓縮在一個 64bit 的 word 裡面，而經過驗證後，也可以發現他有不錯的 encoding/decoding 速度，而由於 simple 系列的演算法是把 integer 盡可能用越少的 bit 去壓縮，例如 1 或是 0 就可以用 1 bit 去表示，這比 Variable Byte 的演算法在壓縮率上面更有優勢，而實作上面也不會太困難，這也是為什麼現在那麼多主流的 database 選用 simple8b 的原因。

simple8b 的 64 bit 中，後面 60 個 bit 拿來放 data，前 4 個 bit 會被拿來當作 selector，決定有多少的 integer 可以被壓縮在 64 bit 的 word 中，其中的 encoding mode 可以參考下表:

| selector         | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12| 13| 14 |15|
|----------------- |-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|-|
| integer coded    | 240 | 120 | 60|  30| 20 |15 |12| 10| 8 |7| 6| 5 |4 |3 |2 |1|
| bits per integer |0 |0 |1 |2 |3 |4 |5 |6 |7 |8 |10| 12| 15| 20| 30| 60|

舉個例子，如果我們要存 [1, 2, 1000, 3, 4, 5, 6, ...] 這個陣列，透過 simple8b，可以看出我們最大的值是 1000, 對應到 2^10，代表我們最多只能 pack 6 個值進入到 64 bit 的 word 中，也就是 [1, 2, 1000, 3, 4, 5], 則後面的 6 且其他數值會被 pack 到後面的 word。

一般來說，simple8b 的 encoding 速度會比 Variable Byte 來得久，不過 decoding 上面應該都會比 Variable Byte 來得好，而壓縮率則是大勝 Variable Byte。

### Binary Packing (BP128)

Binary Packing 是藉由判斷給定的 array 中最大的數字，需要多少 bit 去存，而將其他數字都用同一種 bits 數量去儲存的方法，例如下面這個範例:

```
2, 3, 5, 6, 8, 7, 7, 7
```

其中最大的數字 8，需要 3 bits 去儲存，這樣我們也把其他的數字也用 3 bits 去 encode，原本一個 integer 需要 32 bits，這樣一來一個 integer 就只需要 3 bits，而 binary packing 還有一個特性是把一組的數字一起壓縮，像是著名的 BP128 講的就是把 128 integer 一起透過 fast bit packing 去做壓縮，如果一個 integer 需要 b 個 bits，這樣總共就需要 128*b bits 就夠了，如果 total 數字沒辦法被 128 整除，就可以透過補零的方式。

另外是 binary packing 牽扯到 bit packing 怎麼實作的部分，在這篇論文[Decoding billions of integers per second through vectorization](https://arxiv.org/pdf/1209.2137.pdf) 和作者的 [blog](https://lemire.me/blog/2012/03/06/how-fast-is-bit-packing/) 中也有討論，另外是作者的 C++ 實作，只要透過 shift and mask 就可以將不同長度的 bit 壓縮在不同 bytes 裡面，所以壓縮解壓縮的速度非常快，另外是也可以實作出 SIMD 的版本 SIMD-BP128，印象中論文裡面是寫壓縮解壓縮率會比 scalar 的快上一倍。

整體而言 BP128 不管在壓縮率、encode/decode 速度上面都很不錯，也是在 benchmark 中常常被拿來比較的對象。


### Frame of reference (FOR)

以上的演算法在壓縮前，都是透過 Differential coding， 也就是算 delta 的算法，讓要壓縮前的資料變小，不過這樣的處理方式有個壞處是，沒辦法辦到想快速搜索值，一定都得要把資料先解壓縮回來，而 Frame of reference 的作法可以解決這類的問題。

Frame of reference 基本上是來自這篇 1998 年的論文 Jonathan Goldstein, Raghu Ramakrishnan, and Uri Shaft. 1998. Compressing Relations and Indexes. In Proceedings of
the 14th International Conference on Data Engineering. 370–379.

而這邊的 frame of reference 講的就是找到一個參照系，frame 跟上面提到的 BP 很類似就是 a sequence of integers with the same bit width。

Frame of reference 講解主要我是參考 Lemire 寫的這篇[文章](https://lemire.me/blog/2012/02/08/effective-compression-using-frame-of-reference-and-delta-coding/)，Lemire 出了很多 paper 在討論這個議題，文章內容都寫得很棒，很適合大家去爬。

用例子來說明比較快，給定下列的數列

```
107, 108, 110, 115, 120, 125, 132, 132, 131, 135
```

每個數字可以用 8 bits，總共需要 80 bits 才能存下全部的數字。frame-of-reference 就是要找出陣列的 range 和最小的數字，這邊我們可以看到是 107 到 135，接著我們對這個陣列減去 107 看看:

```
0, 1, 3, 8, 13, 18, 25, 25, 24, 28
```

再看一次會發現，現在我們一個數字只需要用 5 bits 就存得下了，不過我們當然還是要使用 8 bits，去存下一開始的 reference 107，還需要另外的 3 bit 的 metadata，來紀錄我們目前的資料長度是 5 bits，這樣總共是 8+3+95=45 bits，比起原本的 80bits 省下不少空間，另外是搜索上面也有幫助，像是你要搜索 1000 這個數字，很快的比對 107 和資料長度 5bits，可以馬上知道這個 block 裡面，並沒有 1000 這個數字，對於我們 search 上面也有幫助。

#### FOR variant 
而 FOR 還有一些變形，像是上面這個數列，還可以透過之前介紹的 delta 的方法進一步壓縮，變成

```
1, 2, 5, 5, 5, 7, 0, -1, 4
```

現在我們知道除去 -1, 我們可以用 3bits 把東西存下來，但是這個 -1 還是很討厭，幸好還有一個有名的演算法叫做 [zigzag](https://developers.google.com/protocol-buffers/docs/encoding)，可以將負數都 encode 成正數，在 google 的 protocol buffer 裡面也有用到。

除了 Delta，另外一種做法是可以透過 XOR，也可以把數字變小，而且不會產生負數。

#### Patched coding

其實有件事在 FOR or BP 裡面都可能會遇到的是，像下面的 integer array 處理不好的狀況

```
1, 4, 255, 4, 3, 12, 4294967295
```

因為裡面有個超大的數字，變成每個數字又要用回 32bits 去存，那究竟我們有沒有什麼方法可以去改善呢，這邊就有人提出了一個 Patched coding 的方法，這也是常常在論文中看到的 PFor，意思就是決定一個 b bits width 去壓縮，而大於 2^b bits 的數字當作 exception 放在其他的 page 裡面，這邊有蠻多不同的方法去實作這段，詳情還是要看 paper 或是 implementation 如何實作的，不同的實作也是會對不同的測資有影響。


## 結論


在看了幾個資料[1](https://people.csail.mit.edu/jshun/6886-s19/lectures/lecture19-1.pdf), [2](https://people.csail.mit.edu/jshun/6886-s20/lectures/lecture19-1.pdf) 後發現，壓縮 integer 這擋事不外乎上面講的這幾種方式，不過在 SIMD 慢慢流行下，這些演算法也重新被改良，希望能夠用 SIMD 去加速，這的確是我們蠻樂見的，也有看到 influxdb 想要在 timestamp 壓縮這段，可以從 simple8b 改良成用 SIMD-PFor，這也是因為 SIMD 也漸漸是標準配備了，然後 github 上面可以找到一堆開源的實作，像是 [TurboPFor-Integer-Compression](https://github.com/powturbo/TurboPFor-Integer-Compression) 、[simple8b](https://github.com/jwilder/encoding/tree/master/simple8b) 、[BP128](https://github.com/lemire/bp128) 和 [FastPFor](https://github.com/lemire/FastPFor)，有需要的可以直接拿來套用結果看看。

而網路上也有幾篇文章蠻值得參考的:
- https://michael.stapelberg.ch/posts/2019-02-05-turbopfor-analysis/
- https://lemire.me/blog/2012/02/08/effective-compression-using-frame-of-reference-and-delta-coding/
- https://lemire.me/blog/2012/03/06/how-fast-is-bit-packing/

## Reference
- [Decoding Billions of Integers Per
Second Through Vectorization](https://people.csail.mit.edu/jshun/6886-s19/lectures/lecture19-1.pdf)
- [Techniques for Inverted Index Compression](https://arxiv.org/pdf/1908.10598.pdf)
- [A General SIMD-based Approach to Accelerating Compression
Algorithms](https://arxiv.org/pdf/1502.01916.pdf)
- [Decoding billions of integers per second through vectorizatio](https://arxiv.org/pdf/1209.2137.pdf)
- [Upscaledb: Efficient Integer-Key Compression in a Key-Value Store using
SIMD Instructions](https://core.ac.uk/download/pdf/79463992.pdf)

<span>Photo by <a href="https://unsplash.com/@jjying?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">JJ Ying</a> on <a href="https://unsplash.com/s/photos/compress?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
