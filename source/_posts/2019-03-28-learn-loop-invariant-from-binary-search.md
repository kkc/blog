---
title: 透過 loop invariant 學習怎麼寫正確的 binary search 
catalog: true
date: 2019-03-28 17:12:32
subtitle: 不容易，怎麼多年後才懂
header-img: egg.jpeg
tags: algorithm
---

## Preface

Binary search 記得是我剛入門寫程式的時候，前幾個回家作業，當時寫出來時，覺得整個程式就很直覺，對這個也不太有什麼疑問，直到最近看到 Programming Pearls 這本書裡面，有寫到大概 90% 的 binary search 都是錯誤的，甚至第一版的 binary search (1946 的版本)，直到 1962 年才發現有 Bug。

> I’ve assigned this problem in courses at Bell Labs and IBM.  Professional programmers had a couple of hours to convert the above description into a program in the language of their choice; a high-level pseudocode was fine.  At the end of the specified time, almost all the programmers reported that they had correct code for the task.  We would then take thirty minutes to examine their code, which the programmers did with test cases.  In several classes and with over a hundred programmers, the results varied little: ninety percent of the programmers found bugs in their programs (and I wasn’t always convinced of the correctness of the code in which no bugs were found).

> I was amazed: given ample time, only about ten percent of professional programmers were able to get this small program right.  But they aren’t the only ones to find this task difficult: in the history in Section 6.2.1 of his Sorting and Searching, Knuth points out that while the first binary search was published in 1946, the first published binary search without bugs did not appear until 1962.

其實 google 也有一篇[文章](https://ai.googleblog.com/2006/06/extra-extra-read-all-about-it-nearly.html)在探討 binary search，先來看下面這個 binary search 的程式。

```
func Search(input_arr []int, target int) int {
    s := 0
    e := len(input_arr) - 1

    for s <= e {
        m := (s + e) / 2

        if input_arr[m] < target {
            s = m
        } else {
            e = m - 1
        }
    }

    return s
}
```

這個範例明眼人一看就知道 `m := (s + e) / 2` 會有溢位的問題，而通常會有兩種改法:

1. `m := s + (e - s)/2`
2. `m := int(uint(s+e) >> 1)`

但是除了這個之外，其實我寫的這個例子還有其他問題，最主要的就是 [Off-by-one errors](https://en.wikipedia.org/wiki/Off-by-one_error) 這個問題，如果把 [1,2,3,4] 當作 input，然後 target 為 3 的情況，其實會跑進無窮迴圈：

1. s=0, e=3, m=1 且 input_arr[1] = 2 < 3，所以 s = m
2. s=1, e=3, m=2 且 input_arr[2] = 3 >= 3 ， 所以 e = m - 1
3. s=1, e=1, m=1 此時這個程式，因為一直維持 s <= e 就會跑進無窮迴圈

而這個邊界條件，就是要調整 +1, -1 的問題，非常的難搞，這裡有好幾個地方要配合才行

1. e 的邊界是 `len(input_arr)` or `len(input_arr) - 1`
2. s <= e or s < e
3. s = m or s = m + 1
4. e = m or e = m - 1

網路上甚至可以找到範本，專門拿來對付 leetcode 上面的問題，雖然也是有人講可以直接在迴圈中判斷 if input_arr[m] == target 做跳出就行了，但是這樣的寫法顯然無法解決從找出 sorted array 中找出 lower_bound or upper_bound，這就讓我想知道是否有更科學的方法可以幫助我們。

## Loop invariant to the rescue

很幸運的，在網路上找到幾篇文章(都列在 reference 了)幫助我理解怎麼使用 loop invariant 去解決這個問題，我也查了下 Introduction to Algorithm 裡面的 loop invariant 定義:

```
We use loop invariants to help us understand why an algorithm is correct. We must show three things about a loop invariant:

1. Initialization: It is true prior to the first iteration of the loop.
2. Maintenance: If it is true before an iteration of the loop, it remains true before the next iteration.
3. Termination: When the loop terminates, the invariant gives us a useful property that helps show that the algorithm is correct.
```

整個看下來有點歸納法的意味，就是定義一個性質，在 loop 開始前，執行完一次 loop interation，和結束時都可以保證這個性質成立，這樣就可以得到正確的程式結果。

先看看下面這個簡單的例子

```
func find_max(a []int) {
    max = -INF

    for i:=0; i < len(a); i++) {
        if (a[i] > max)
            max = a[i]
    }

    return max
}
```

以這個例子來說，我們的 loop invariant condition 可以設定為 max 總是在給予的 a array 前 i 個元素中，然後去驗證每次跑迴圈的時候，都符合這個條件，就可以確定這個演算法是正確的。

## 透過 loop invariant 寫 binary search

前面提的那個例子，大家一定會覺得有點太簡單，實在不知道對我們寫程式有什麼幫助，接下來透過 binary search 的例子，相信大家可以有更不一樣的感受。

首先要來定義我們的問題:
- Pre condition: 
  在 binary search 中，我們會有一個 sorted list，然後從中找到 target。

  sorted list = [3, 5, 6, 13, 18, 21, 23]
target = 18

- Post condition:
  找出 key 是否在 list 中

而定義 list 的區間其實有四種方法
```
1. A[low] <  A[i] <  A[high]
2. A[low] <= A[i] <  A[high]
3. A[low] <  A[i] <= A[high]
4. A[low] <= A[i] <= A[high]
```

看過許多資料後了解方法二是比較好的選擇， `i ∈ [low,high)`，也就是左閉右開這個方法，也就是右邊的值並沒有包含在這個區間內，其實也是最直覺的方法，這邊很推薦大家看這份知乎的文章: [二分找查有幾種寫法?](https://www.zhihu.com/question/36132386)去了解為什麼要取這個區間，其實我以下很多內容也是看這篇文章而通透的。

而選擇了這個區間後，我們先來個基本版的 binary search 實做，才容易解釋 loop invaraint
```
func Search(input_arr []int, target int) int {
    low := 0
    high := len(input_arr)  // 符合 i ∈ [low,high)

    for low < high {
        mid := low + (high - low) / 2

        if input_arr[mid] == target {
            return mid
        } else if input_arr[mid] < target {  // target 在 mid 右側
            low = mid + 1
        } else {                           // target 在 mid 左側
            high = mid
        }
    }

    return -1
}
```

我們這裡設定的 loop invariant 性質，跟區間很有關係
1. 搜索區間 `[low, high)` 不為空的話，low < high 才會成立，反之為空的話，low == high 會離開迴圈
2. 找出來的 sub range 搜索區間都是 `[low, high)`

有了這些條件後，我們可以分析下迴圈結束的 boundary condition，來先個比較小的測資，來模擬測試區間變小的情況。

### 範例 1

如果我們有個 array 裡面只有一個元素 [0]，然後我們要找的 target 為 1 時，透過以下的 step

1. 我們的初始搜索區間為 [0, 1)，low = 0, high = 1, mid = 0
2. 因為 input_arr[mid] = 0 < 1，所以 low = mid + 1 ，此時 high & low 皆為 1 且重合，搜索區間為空集合，離開迴圈。
3. 回傳 -1 代表這個 array 沒有我們要的值 

已上面這個例子，我們可以得知，如果把跳出的條件寫成 `low <= high` 或是 low 寫成 mid 都會出問題，因為會不符合 loop invaraint ，這邊要理解的就是搜索區間變成空集合在這個程式中，是怎麼表示才是正確的。 

### 範例 2

在了解怎麼離開迴圈後，讓我們再看看比較長的測資，[3, 5, 6, 13, 18, 21, 23]，從中間找 18 這個值

<div style="width: 300px; margin: auto">![example](./binary_search_1.png)</div>

從這個過程中我們可以看到，不管是找右區間還是左區間，我們的 L & H 的移動法則都是要保持搜索區間為 [L, H)，然後慢慢把搜索區間變小。

<div style="width: 300px; margin: auto">![example](./binary_search_2.png)</div>

再看一下這個例子，如果我把 18 改成 19，一樣是搜索 18 這個值，會發現結束時，我們的 low == high 並且跳出回圈回傳1，就跟範例 1 的情況一樣，這時我們的 [low, high) 就成為空集合了。

## 透過 loop invariant 寫 lower bound 

以上我們的 binary search 的例子，只能找出 target 是否在 sorted array 或是不在 sorted array，但是如果要找 lower bound or upper bound 就無法使用了，下面給個例子什麼是 lower bound & upper bound。

```
            upper bound
                +
[0, 1, 2, 2, 2, 3, 4, 5]
       ^
       lower bound
```

如果要找 lower bound 其實就是稍微改寫下我們的 binary search 

```
func Search(input_arr []int, target int) int {
    low := 0
    high := len(input_arr)  // 符合 i ∈ [low,high)

    for low < high {
        mid := low + (high - low) / 2

        if input_arr[mid] < target {  
            low = mid + 1
        } else {                     
            high = mid
        }
    }

    return low
}
```

這邊的 loop invariant 跟之前的很相似，不過有些小變形
1. 搜索區間 `[low, high)` 不為空的話，low < high 才會成立，反之為空的話，low == high 會離開迴圈
2. 找出來的 sub range 搜索區間都是 `[low, high)`
    - 右邊的區間 `[high', high)` 都是 >= target 的值
    - 左邊的區間 `[low, low')` 都是 < target 的值

接著直接看圖說故事:

<div style="width: 400px; margin: auto">![example](./binary_search_3.png)</div>
一樣維持搜索區間為 [L, H) (藍色)

<div style="width: 400px; margin: auto">![example](./binary_search_4.png)</div>
因為 array[mid] >= target，所以走到 H = mid，這裡其實產生了右邊的區間 `[high', high)` (粉色)，我們可以知道這個區間其實有著 >= target 的特性，所以 target 也有可能落在這個區間內，到最後要找答案的時候這個區間很重要。

<div style="width: 400px; margin: auto">![example](./binary_search_5.png)</div>
接著看到 array[mid] < target，這代表了 `[low, mid]` 的這個區間都是小於 target 的，所以我們選擇讓 L = mid + 1，這樣產生出來的 `[low, low'）`的區間 (綠色) 才符合我們所定義的特性，但是可以發現藍色區間還是 `[Low', high')`，我們的目標是要讓藍色區間縮小到不見，並保持 loop invariant。

<div style="width: 400px; margin: auto">![example](./binary_search_6.png)</div>
<div style="width: 400px; margin: auto">![example](./binary_search_7.png)</div>
因為 array[mid] == target 所以繼續拓展右邊的區間，記得這個區間內的值都是 >= target 的

<div style="width: 400px; margin: auto">![example](./binary_search_8.png)</div>
結束時跟之前的例子一樣 L=H 會重合，這邊我們要的答案其實不管回傳 L 或是 H 的 index 都是一樣的結果，但是其實可以想成是取出粉紅色的第一個值，就會是我們要找的 lower bound。


## 心得

其實 binary search 的變化真的很多，但是只要了解自己要搜索的區間長怎麼樣，就比較不會卡來卡去在那邊 +1, -1, 而最後寫的 lower bound 的方法其實也適用於一般的 binary search，可說是比較簡單又不容易錯的版本，不過要了解這個 loop invariant 怎麼定義區間，怎麼移動 low, high 去產生新的搜索區間，我還是建議大家用紙筆自己畫畫看，其實會比較有感覺，也可以拿 `A[low] <= A[i] <= A[high]` 這個為例子看看程式要怎麼寫才對，這篇文章的圖文寫得比較快，如果有不清楚或是錯誤的地方在請大家指正 :)

## Reference

- [binary search and loop invariant](https://www.eecs.yorku.ca/course_archive/2013-14/W/2011/lectures/09%20Loop%20Invariants%20and%20Binary%20Search.pdf)
- [How to write binary search correctly](https://zhu45.org/posts/2018/Jan/12/how-to-write-binary-search-correctly/)
- https://www.zhihu.com/question/36132386
