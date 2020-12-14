---
title: 學習使用 compiler vector extension 去寫 SIMD 程式
catalog: true
date: 2020-11-01 20:09:10
subtitle: compiler vector extension 好棒棒
header-img: simd.jpeg
tags:
  - gcc
  - performance
---

最近強者我 Tead lead [Champ Yen](https://www.facebook.com/champ.yen) 在公司內部做了一次 experience sharing，內容非常的精彩，分享了怎麼使用 compiler vector extensions 去寫 SIMD 的 program，進而將 program 的效率提升，並且可以產出 portable 的 program。

## SIMD 到底是什麼
SIMD 的全名是 single instruction multiple data，而顧名思義就是使用一個 instruction 去操作多組 data。

在 [Flynn taxonomy](https://en.wikipedia.org/wiki/Flynn%27s_taxonomy) 裡面將 information stream 分成了 instruction 和 data，進而對計算機做分類，而普通我們認知的 instruction 操作一個 data (register) 被稱為 SISD，而 SIMD 之所以重要是因為電腦的單核的頻率在古早前就上不去了，詳情可以見下圖

![此圖](https://github.com/karlrupp/microprocessor-trend-data/raw/master/48yrs/48-years-processor-trend.png?raw=true)

而改善程式的效率的方式，就變成探索如何將其變成 parallelism 的過程，這方面就多了如何善用 Multicore，熟悉 NUMA，以及採用 SIMD 之類的技術。

## SIMD 為什麼會比較快

這頁取自交通大學劉志尉老師的課程[投影片](http://twins.ee.nctu.edu.tw/courses/ca_13/lecture/CA_lec08-chpater_4-vector_processing.pdf)， 從中可以看到 scalar code 和 vector code 各自需要的 instruction 數量，而 scalar code 還要考慮外面有個 loop 迴圈，所以整體需要時間更多。

![](https://i.imgur.com/JdA2fyy.png)

## SIMD instruction 有哪些 type

- Load/Store
- Per-Lane
    - Arithmetic
    - Bitwise, Logical
- Cross/Inter Lane
    - Permute, Select, Shuffle(LUT)
    - Alignment
    - Pack & UnPack
- Reduction (e.g Average of vector)
    - Minimum
    - Maximum
    - Average
- Special (特殊的 instruction)
    - NN specific ISA
    - inter-lane + per lane attributes

## 為什麼需要 compiler vector extension

1. 可以使用 vector 去提升程式的 performance
2. 比直接使用特定平台的 intrinsics/ASM 來的容易使用
3. 比較容易透過這種方式，去修改已經存在的 C/C++ 程式
4. portability (大加分)
5. 可以跟 OpenMP 一起使用 (這邊我其實沒很懂，因為沒寫過 openmp)

## 如何使用呢?

### GCC vector type declaration

先來學如何宣告 vector，可以使用下列語法

```cpp=
typedef SCALAR_TYPE TYPE_NAME __attribute__((vector_size(SIZE), aligned(1)));
```
e.g:
```cpp=
typedef int v4si __attritube__ ((vector_size (16), aligned(1)));
typedef float v4sf __attribute__ ((vector_size (16));
typedef double v4df __attribute__ ((vector_size (32)));
typedef unsigned long long v4di __attribute__ ((vector_size (32)));
```

以上的宣告很簡單，以 v4si 為例，我們宣告了一個 vector_size 為 16bytes 的 vector，其分割成 4個 int sized unit。我們可以用下列的方式去初始化他們

```cpp=
v4si a = {1,-2,3,-4};
v4sf b = {1.5f,-2.5f,3.f,7.f};
v4di c = {1ULL,5ULL,0ULL,10ULL};
```
### 用操作 scalar 的方式使用 SIMD

```cpp
typedef int v4si __attribute__ ((vector_size (16)));

int main() {
    v4si a = {1,2,3,4};
    v4si b = {3,2,1,4};
    v4si c;

    c = a + b;      /* The result would be {4, 4, 4, 8}  */
    c = a > b;     /* The result would be {0, 0,-1, 0}  */
    c = a == b;     /* The result would be {0,-1, 0,-1}  */
}
```

## 再來學下其他的 Compiler Build-in function

- `__builtin_shuffle`
- `__builtin_convertvector`
- `__builtin_prefetch`
- `__builtin__clear_cache`

### 舉個例子

這是我給 Champ 出的問題，如何高效地把一個 `vector<int>` 轉成 `vector<float>`，這邊就使用了 `__builtin_convertvector`，主要是因為 vector 也是連續的記憶體操作，所以可以使用 pointer 指過去後使用 SIMD 操作。

```cpp
typedef float v8sf __attribute__ ((vector_size (32)));
typedef int v8si __attribute__ ((vector_size (32)));

std::vector<int> vint(TEST_LEN)
std::vector<float> vfp(TEST_LEN)

srand(time(NULL))
for(int i=0; i < TEST_LEN; i++) {
    vint[i] = rand()
}

int *intp = vint.data();
float *fpp = vfp.data();
struct timeval stime, etime;
gettimeofday(&stime, NULL);
for(int i=0; i+8 < TEST_LEN; i+=8) {
    *((v8sf*)(fpp+i)) = __builtin_convertvector(*(v8si*)(intp + i), v8sf);
}
gettimeofday(&etime, NULL);
```

## Architecture Dependent Compiler Intrinsics

可以使用 union 去撈出 vector register 裡面個別的值，這樣對於 debug 或是真的要轉型就不會那麼麻煩。

### gcc:
```cpp
#include <immintrin.h>

typedef unsigned char u8x16 __attribute__ ((vector_size (16)));
typedef unsigned int u32x4 __attribute__ ((vector_size (16)));

typedef union {
    __m128i mm;
    u8x16   u8;
    u32x4   u32;
} v128
```

### LLVM/clang:
- use vector extension variables directly 

Ref: https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html

## 其他的 Tips

### Porting & Troubleshoot 的一些方法

需要人工算一下將原本的 loop 切成 fixed size 的 chunk (e.g 8 for int32_t)，接著再把 loop 內部換成 vector operations。

### Deployment - Function Multi-Versioning (a.k.a FMV)

- Pros:
透過這個方法可以讓編譯出來的 binary 跑在不同的平台上面

- Cons:
binary 會變肥大

```cpp=
__attribute__((target_clones("avx2", "avx", "sse4.2", "sse3", "sse2", "default")))
int main(void) {
    v8si v0 = {0, 1, 2, 3, 4, 5, 6, 7};
    v8si v1 = {8, 9, 10, 11, 12, 13, 14, 15};
    
    v8si v2 = v0 + v1;
    return v2[3];
}
```

有興趣的人可以透過 https://godbolt.org/z/of5d6v 去看看有加這行，會多產生不同的 assembly code，這樣一來就對應不同平台上面的 vector operations。

Ref: https://lwn.net/Articles/691932/

## 使用 SIMD/Vector 的一些眉眉角角

- 需要找到 Parallelism 的演算法
- 可以透過不同的方式使用 SIMD, 不過要考慮 portability 的問題。
- unsupported operations
    - division, high level function(math functions)
- Floating point
    - cross device compatibility
- Boundary handling 等邊界問題
    - 需要使用 padding, predication, 或是 fallback 去使用 scalar
- Divergence
- Register splitting
- 需要考慮 Non-Regular Access/Process Pattern 還有 dependency
    - 像是 LUT, AoS (Array of structure)

## 一些心得

通過 Champ 這個分享，我真的終於知道如果安全的使用 SIMD，之前都是看一堆 project 寫 x86 assembly 寫得很爽，或是只能依靠 compiler 的 [Automatic vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization )，現在終於知道也可以透過 compiler instrinsic 來寫，另外是 Champ 也提到 SIMD 這個技術已經發展了很多年，而 compiler instrinsic 像是 gcc 也是從 3.1 就開始支援了，所以大家放心的使用，然後害怕的 portability 的問題也是被解決的蠻好的，而我大概查了一下，如果真的想達成 compiler agnostic，也可以使用 libarary instrinsic，不過就各有優缺點了，用 compiler instrinsic 的好處，整體的程式碼還是可以寫得跟處理 scalar 一樣，個人也覺得看起來蠻舒服的。

另外是在搜索相關資料的過程中，看了很多不錯的文章，像是 stackoverflow 的 blog 就有提到一些 SIMD 的[應用](https://stackoverflow.blog/2020/07/08/improving-performance-with-simd-intrinsics-in-three-use-cases/)，但也從這個[快快樂樂SIMD](https://www.slideshare.net/WeiTaWang/simd-109492525) 看到蠻多要注意的地方，而像是 AWS 所提供的 x86 & ARM 機器也都會有提到，他們各自支援的 SIMD 指令，我們如果真的要學習榨效能，這塊的基本概念真的也需要撿起來。

## Reference
- https://gcc.gnu.org/onlinedocs/gcc/Vector-Extensions.html
- https://www.karlrupp.net/2015/06/40-years-of-microprocessor-trend-data/
- https://github.com/karlrupp/microprocessor-trend-data
- https://www.uio.no/studier/emner/matnat/ifi/IN3200/v19/teaching-material/avx512.pdf
- https://champyen.blogspot.com/2020/05/clang-gcc-vector-extension.html
- [交通大學 Computer Architecture Lecture 8: Vector Processing](http://twins.ee.nctu.edu.tw/courses/ca_13/lecture/CA_lec08-chpater_4-vector_processing.pdf)
- [快快樂樂SIMD](https://www.slideshare.net/WeiTaWang/simd-109492525)

photo credit from https://unsplash.com/photos/Uf-c4u1usFQ
