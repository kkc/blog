---
title: Golang 的 string interning 技巧
catalog: true
date: 2020-12-14 21:49:23
subtitle: 
header-img: interning.jpg
tags:
  - golang
---

## String Interning

最近在 twitter 上面看到一篇推文

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">Hacked string interning profiler for <a href="https://twitter.com/hashtag/golang?src=hash&amp;ref_src=twsrc%5Etfw">#golang</a>:<a href="https://t.co/EB2uJwzvtx">https://t.co/EB2uJwzvtx</a><br>Allows to understand where to use interning &amp; exact savings for heap size/garbage rate. May be useful for larger projects.<br>Yay or nay? <a href="https://t.co/71h6tNJHaX">pic.twitter.com/71h6tNJHaX</a></p>&mdash; Dmitry Vyukov (@dvyukov) <a href="https://twitter.com/dvyukov/status/1337847667125313538?ref_src=twsrc%5Etfw">December 12, 2020</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

具體在討論這個 [CL](https://go-review.googlesource.com/c/go/+/277376) 要加入 string interning profile，而利用這個可以測量是否該加入 string interning，原本不知道這個是幹嘛的，後來看了一下 comments 內的一些解釋和文章，才知道原來 String interning 是個可以拿來有效減少 memory 使用量的技巧，原理相當簡單，而在其他語言裡面也有這種東西，像是 python 在一些小的數字和文字上面，都是會指向同一組記憶體，藉此來減少 memory allocation 的時間和用量，而 string intening 也是這類技巧的名字(https://en.wikipedia.org/wiki/String_interning)。

## Golang 裡面的運用

在 golang 裡面在什麼地方被用到，也可以看下面幾篇的解釋，個人覺得已經很清楚
- https://artem.krylysov.com/blog/2018/12/12/string-interning-in-go/
- https://commaok.xyz/post/intern-strings/

基本上要做實驗可以透過下面的 function 去看 string 的 pointer 位置，看是不是同一份
```golang
func pointer(s string) uintptr {
    p := unsafe.Pointer(&s)
    h := *(*reflect.StringHeader)(p)
    return h.Data
}

func main() {
    b := []byte("hello")
    s := string(b)
    t := string(b)
    fmt.Println(pointer(s), pointer(t))
}
```
相關的 playgound 連結在[這邊](https://play.golang.org/p/oyq6Pz79EGa)，從這個實驗中，我們可以很快發現，兩個字串指向的記憶體不同，而在 golang 裡面比對 string 如果是同一個 pointer 而且 Len 都一樣，就可以加速比對的過程，而不用一個 byte 一個 byte 的去比較，所謂的 interning，也就是如果我們能夠不重複 allocate 記憶體，都用同一個字串。

* 而有另外一個有趣的點是，這個 playground 裡面可以把 hello 改成 h, 可以看到 pointer 都指向同一個位置，這是透過 compile time constant 的結果，具體可以看這個[CL](https://go-review.googlesource.com/c/go/+/97717/)的解釋，透過 benchmark 提前先 allocate 一個字元在效率上面可以提升很多。

## 自幹 string interning

而在 golang 裡面其實並沒有很多地方有做 interning，但是我們在處理一些資料的時候，其實有機會用到這個技巧，像是第一篇文章裡面有寫到，如果我們要處理大量的 text，如果沒有 interning，可能就需要 allocate 很大量的記憶體去儲存這些資料，另外是像從資料庫讀取東西時，也可以有些數據是一直重複出現的，這時也可以應用同樣的技巧。

而一般來說透過類似 cache 的方式可以自幹 string interning，像是第二篇文章的 code
```golang`
func intern(m map[string]string, b []byte) string { 
    // look for an existing string to re-use 
    c, ok := m[string(b)] 
    if ok { 
        // found an existing string return c 
    } 
    // didn't find one, so make one and store it 
    s := string(b) 
    m[s] = s return s
}
```

但是麻煩的地方跟自己維護 LRU cache 一樣，如果有不需要用到的字串需要被 evict 掉，要不然也會佔據記憶體，另外是要做一個能夠 concurrent 存取的 LRU cache 也是不容易，所以這個東西沒處理也會變成反效果，而大家都會談論到這個 project [https://github.com/josharian/intern](https://github.com/josharian/intern)，裡面是用 sync.map 實作 interning，但是還是要斟酌一下是不是真的該用。

順帶一提是透過這個 CL，我又追到這個 [CL](https://github.com/golang/go/issues/32779#issuecomment-731623919)，裡面的討論也蠻不錯的，也讓我了解到設計系統需要考量的一些東西，我蠻建議看下 dsnet 對做 memorize stirngs during decode 的一些看法和實驗，很多地值得借鑒，像是
  - String 越長，對於 cache 越不友善，有可能 cache hit rate 會下降的很快，而太長的 sring 也需要花更多時間去比對和做 hash，所以只需要選擇 cache 小一點的資料(e.g < 16B)
  - Go men allocator 其實速度很快，配置 16B 的字串只需要 35ns，所以 cache 的 lookup 和比較應該要快於 35ns 才有賺
  - JSON 的 object name 有 high degree of locality，所以有個很小的 cache 去存這段東西很值得
  - 而 JSON 的 value 可能就沒那麼值得去 cache，因為 locality 不好

## 結論

總的來說，string interning 也許不是很常會使用的技巧，但是如果真的有極端的 case，也許拿來使用減少記憶體和 gc 壓力也是不錯的途徑，當然前提還是要經過縝密的 benchmark。

## Reference
- https://commaok.xyz/post/intern-strings/
- https://github.com/bserdar/go/commit/9829fb1b5501df91a44cb24e554310a6f28f123c

<span>Photo by <a href="https://unsplash.com/@jessedo81?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">jesse orrico</a> on <a href="https://unsplash.com/s/photos/storage?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
