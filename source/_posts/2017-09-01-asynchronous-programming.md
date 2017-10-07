title: 淺談 Asynchronous Programming
date: 2017-09-01 22:08:31
tags: programmming
---

<iframe src="//www.slideshare.net/slideshow/embed_code/key/2OtC3FTsUFvtNN" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> 

利用這篇文章稍微紀錄一下，最近在公司分享的淺談 Asynchronous programming 的心得，其實在現在各個網路公司盛行的時代，不懂點 Asynchronous model 應該是不太可能的，而且就算不懂實際上大家也是天天在用，類似寫 Browser 端的 javascript，或是不論 nodejs 或是 python (像是 gevent)，都常常看到大家再談 Asynchronous model 對 performance 帶來的好處，正好同事們很多人正在學 javascript，而有些人似乎不知道最初是在解決什麼問題，所以才會有這個投影片的誕生，不過因為太多方法可以處理 Asynchronous model，我並沒有講得太深入，很多地方只有點到為止，希望之後有機會能夠再補充一些。

所以到底 Asncyhronous or concurrency 解決了什麼問題呢？一言以敝之，追根究底就是消除等待。
concurrency 主要拿來跟 parallellism 比較，concurrency 談的其實就是能夠在同一時間完成很多事情，我喜歡拿做菜為例子，就算只有一個廚師，他還是可以在同時間完成切菜，準備醬料，煮菜等等工作，他會在中間切換來切換去，而不會等到一盤菜好了，再去準備接下來的事情，而 Parallellism 比較像是同時有很多 worker 做差不多事情。

我們一般寫的程式，其實大部分都在處理這種問題，像是 GUI 程式，使用者按了一個 button 後，你不可能完全卡在那邊等待其他的程式跑完，又像是你的 API server 經由網路呼叫一個 3rd party 的外部程式，如果你在那邊傻傻的等，是不是浪費了很多 CPU resource。這時候我猜大家都很聰明想到可以直接用 thread 去操作，避免掉 main process 被卡住的情況，而其實 thread 也算是一種 aysnchronous programming 的 model，而且現今的程式語言也有更好語法去使用它們，像是用 Future or Promise。

在瞭解完要解決什麼問題後，需要知道的是不同解法之間的差異，因為公司同仁大部分都是寫 javascript 或是 python，所以在 asynchronous flow 上面比較多著墨 callback, eventloop, coroutine 還有最後衍生出來的 async/await，而 javascript 天生就使用一個 thread 去達成 concurrency，這點感覺有點奇妙，所以需要先補充一些 linux IO 的知識，像是 blocking IO/non-blocking IO 的差別，最後導出 IO multiplexing 才能講得下去 event loop 怎麼實作的，nginx 之類的 service 就是利用了 IO multiplexing 才有效的解決 [C10K](https://en.wikipedia.org/wiki/C10k_problem) 的問題，使用一個 thread 處理 network IO 請求，其實就減少了增減 thread 的開銷，memory 的使用量也大減，但取而代之的就是，程式會有點難寫難讀，所以就有了 libev, libevent, libuv 這類 library 幫忙處理 asynchronous IO 這部分，使用這類的 lib，上層 program 很簡單就可以使用 callback function 與之互動，等到 socket 的 file descriptor 被 trigger 時再去呼叫 callback 繼續處理下去。

在有了 callback 後，世界並不是就太平了，很快就有人發現可以寫出 callback hell 這種程式，asynchronous progrmamming (主要談 callback) 到這邊就變成語法的改進了，希望能夠把程式寫得更漂亮更易讀一點，所以就有了 promise 或是 coroutine 的方法與其結合，在 coroutine 中會把控制權從 function 中切換回 main task 中，某種程度跟 eventloop 就很相似，所以可以利用 coroutine 的特性加上 non-blocking I/O 成為更好的框架，而程式也會變得像是 synchronous 的樣子，不過要注意一但有程式是 blocking IO 或是 cpu intensive 的任務，就會把這個 thread 卡住，最後提到的 async/await 只不過是語法的變形，其實跟 coroutine 的概念是很相似的，可以讓整個程式更好寫易讀，然後背後又能高效率的處理 IO bound 問題。

最後來總結一下，其實這些方法都是為了解決一些任務太慢而產生的，在我們當使用者的時候(caller)，實際上也許不知道背後這些 async module 是如何運作的，除非是自己要一手從下到上包辦，但還是有些點需要注意，如果這類程式只是處理 networking IO 的話，應該是不會有太大的問題，但如果中間有個 cpu intensive 的任務最好還是要能 fork 出 process/thread 去處理，或是利用 queue 丟給其他的 worker 去處理，所以一旦我們清楚這些架構後，才能知道採取哪種方式處理問題是比較好的。


Reference
1. [What the heck is the event loop anyway](https://www.youtube.com/watch?v=8aGhZQkoFbQ) 非常棒的 javascript eventloop 講解
2. [javascript-coroutine](https://x.st/javascript-coroutines/)
3. [concurrency-vs-multi-threading-vs-asynchronous-programming-explained](https://codewala.net/2015/07/29/concurrency-vs-multi-threading-vs-asynchronous-programming-explained/)
