---
title: FFmpeg libav decode 筆記
catalog: true
date: 2019-01-12 11:24:06
subtitle: 一些基本知識
header-img: thomas-william-250918-unsplash.jpg
tags: ffmpeg
---

# Preface

身處在一間做 Surveillance 的公司，一定要熟悉下 FFmpeg 怎麼使用，FFmpeg 真的是蠻偉大的，應該全世界大部分需要處理影音的公司都對他不陌生，FFmpeg 是一個跨平台免費又開源的影音處理方案，採用 LGPL 或是 GPL 的 License，單純使用 ffmpeg 或是 ffprobe command 就可以做到很多加解碼轉檔等等的事情，非常的方便！

FFmpeg 也有提供 library 給開發者呼叫並且整合在自己的程式中，這篇筆記基本上記錄了下怎麼使用它來做基本的解碼，FFmpeg 包含以下幾個 lib (只列了幾個我常用的)

Libavcodec: encode/decode 的 framework 包含很多影音的加解碼器
Libavformat: 對 video 的封裝
Libswscale: 圖像縮放，顏色空間轉換

# Intro

基本上這篇心得，很多來自這篇文章[https://github.com/leandromoreira/ffmpeg-libav-tutorial](https://github.com/leandromoreira/ffmpeg-libav-tutorial)的內容，非常推薦想要學 FFmpeg 的同學作為參考，算是我找過網路上寫的最平易近人的說明。而這篇文章主要會紀錄 video 相關的心得，因為我對音訊還沒那麼熟。

# Video

Video 其實可以視為一堆圖片的集合，小時候都有玩過一種東西，就是一本書上面有很多圖，在快速翻動的時候，就能感覺到上面的東西在移動。

# Codec

Codec 的工作就是把資料縮小，這邊給個概念，如果我們把數以百萬計的圖片放進一個電影檔裡面，那這個檔案勢必非常的大，來做個簡單的數學：

讓我們拿個高清的影片，解析度為 1080 x 1920，然後每個 pixel 都用 3 bytes 去記錄他的顏色資訊 (24 bit color，這裡可以表達 16,777,216 不同的顏色，我們的眼睛好厲害)，這個影片是 24 fps (frame per second)，然後長度為 30 分鐘，我們做個簡單的數學計算:

```
toppf = 1080 * 1920 //total_of_pixels_per_frame
cpp = 3             //cost_per_pixel
tis = 30 * 60       //time_in_seconds
fps = 24            //frames_per_second
```

需要的儲存空間 = `tis * fps * toppf * cpp`

簡單的計算後發現，這個電影檔居然要花我們 `250.28GB` 的空間，還有 `1.11Gpbs` 的流量，這也是為什麼我們需要 codec 來幫助我們壓縮檔案。

# Container

Container 可以視為一個 wrapper，裡面包含了不同的 stream (通常是 video 和 audio)，然後這個 container 通常也提供了 Metadata 像是 video title , resolution 之類的資訊，

這邊加點我個人的筆記，mp4 和 mpeg4 video data，就是一種 container 和 stream 的概念，也是瓶子和內容物，mp4 這個瓶子一般來說都是裝 standard mpeg4 video codec 的資料，但是如果硬拿來裝其他的東西也可以，只是應該沒人會這樣做。

# FFmpeg Libav Architecture

要知道怎麼 encode/decode 得先了解下 libav 的 architecture

![img](flow.png)

1. 讀取 media file 到 `AVFormatContext` 這個 compoenent 裡面，基本上這個動作其實只會讀取檔案的 header 而已，而經由這個 header 我們可以知道這包 container 裡面有多少 stream 。
2. 如果我們要讀取 Container 裡面的 stream 的話，libav 會把它封裝在 `AVStream` 這個 component 內，我們就可以經由這個 component 讀取到 stream 的資料。
3. 假設我們的 Container 裡面有兩個 stream ，一個是 video (encoded by H264 CODEC) 另外一個是 audio (encoded by AAC CODEC)，我們可以從中讀取一小段資料進 `AVPacket` 這個 Compoenent。
4. 資料在 `AVPacket` 中還是被 *encode* 的狀態，這時候我們會需要 `AVCodec` 的幫忙，將 packet 裡面的資料 decode 出來到 `AVFrame` 中，我們就可以拿到 uncompressed frame。

## Detailed Decoding Flow

主要的範例程式可以參考[hello_world.c](https://github.com/leandromoreira/ffmpeg-libav-tutorial/blob/master/0_hello_world.c)，接下來我會對其中比較重要的幾個 step 做個簡單的筆記。

1. 一開始要先對 `AVFormatContext` 這個結構配置記憶體，經由這個結構我們才能得到 container 的 format。

    ```
    AVFormatContext *pFormatContext = avformat_alloc_context();
    ```

2. 接著使用 `avformat_open_input` 去將檔案讀進到我們之前配置好的 `AVFormatContext`，這個 function 最後有兩個 arguments，第一個是 `AVInputFormat`，傳入 `NULL` 他會自動猜測格式，第二個是 `AVDictionary` 通常是拿來配置 demuxer 的參數。

    ```
    avformat_open_input(&pFormatContext, filename, NULL, NULL);
    ```

3. 讀進 `AVFormatContext` 後，可以印出 container 的 format 和 duration。 

    ```
    printf("Format %s, duration %lld us", pFormatContext->iformat->long_name, pFormatContext->duration); 
    ```

4. 使用 `avformat_find_stream_info` 拿來讀取 media file 裡面的 data ， 在呼叫完 `avformat_find_stream_info(pFormatContext,  NULL);` 這個方法後， 才能從 `pFormatContext->nb_streams` 裡面得到 context 有多少個 stream，接著可以用 pFormatContext->streams[i] 得到不同的 stream (`AVStream`)。

    ```
    for (int i = 0; i < pFormatContext->nb_streams; i++)
    {
      // pFormatContext->streams[i]
    }
    ```

5. 因為每個 stream 都有可能是用不同的 codec 壓縮的，我們可以經由 `AVCodecParameters *pLocalCodecParameters = pFormatContext->streams[i]->codecpar;` 從每個 stream 中取得對應的 `AVCodecParameters`

6. 利用剛剛取得的 parameter 和 `avcodec_find_decoder` function 找到對應的 `AVCodec`

    ```
    AVCodec *pCodec = avcodec_find_decoder(pLocalCodecParameters->codec_id);
    ```

7. 取得 Codec 後，我們需要配置記憶體給 `AVCodecContext`，這個結構是等等要拿來 encode/decode 用的，我們另外還需要將 codec 的 parameter 也複製到這個 context 中，在我們配置好 codec 的 context 後，還需要使用 `avcodec_open2` 才能真的在之後使用這個 context。 

    ```
    AVCodecContext *pCodecContext = avcodec_alloc_context3(pCodec);
    avcodec_parameters_to_context(pCodecContext, pCodecParameters);
    avcodec_open2(pCodecContext, pCodec, NULL);
    ```

8. 我們需要把 packet (`AVPacket`) 從 stream 讀出來後，然後 decode 成一張張的 frame (`AVFrame`)

    ```
    // 配置記憶體
    AVPacket *pPacket = av_packet_alloc();
    AVFrame *pFrame = av_frame_alloc();
    ```

    這邊我們要使用 `av_read_frame` 這個 function 將 video_streaming 的資料從 `AVFormatContext` 中讀出來到 packet 中

    ```
    while (av_read_frame(pFormatContext, pPacket) >= 0) {
        avcodec_send_packet(pCodecContext, pPacket);
        avcodec_receive_frame(pCodecContext, pFrame)
        //...
    }
    ```

    接著將 raw packet (compressed frame) 送進 decoder，然後把 raw data frame (uncompressed frame) 取出來，這兩組 API `avcodec_send_packet` & `avcodec_receive_frame` 需要互相搭配使用，使用 `avcodec_send_packet` 將 packet 送到 `AVCodecContext` 中，然後透過 `avcodec_receive_frame` 將解碼後的 frame 拿出來，然後要注意一點是 `avcodec_send_packet` 和 `avcodec_receive_frame` 並不一定是一對一的關係，有時候需要多送幾個 packet 讓 `AVCodecContext` 緩存幾張 frame 的 data，而這邊拿到的 `pFrame->data` 的格式是 (YCbCr)[https://en.wikipedia.org/wiki/YCbCr]，如果想要轉成 RGB 的話還需要使用到 `SwsContext` 之類的方法。

9. 接著我們就可以印出取得的資訓像是 frame_number 或是 pts 等等印出來驗證摟

    ```
    printf(
        "Frame %c (%d) pts %d dts %d key_frame %d [coded_picture_number %d, display_picture_number %d]",
        av_get_picture_type_char(pFrame->pict_type),
        pCodecContext->frame_number,
        pFrame->pts,
        pFrame->pkt_dts,
        pFrame->key_frame,
        pFrame->coded_picture_number,
        pFrame->display_picture_number
    );
    ```

針對 AVPacket 和 AVFrame，我另外畫了一張流程圖

![flow2](./flow2.png)

# 心得

ffmpeg 經過多次改版，API 和文件其實變得比較人性化一點，網路上也不像之前資料那麼少了，經由這次的練習也找到了不少有用的資料，都列在底下的 Reference 供大家參考，如果有哪邊寫錯的，還請大家指正了。

# Reference
1. [ffmpeg-encoding-course](http://slhck.info/ffmpeg-encoding-course/)
2. [https://github.com/leandromoreira/ffmpeg-libav-tutorial](https://github.com/leandromoreira/ffmpeg-libav-tutorial)
3. http://leixiaohua1020.github.io/#ffmpeg-development-examples
4. https://ffmpeg.org/doxygen/4.0/group__lavf__decoding.html#details
5. https://ffmpeg.org/doxygen/4.0/group__lavc__decoding.html 
