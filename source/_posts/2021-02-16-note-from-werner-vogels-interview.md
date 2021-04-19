---
title: Werner Vogels 談 Amazon 怎麼從 Monolith 到 Service-based Architecture
catalog: true
date: 2021-02-16 21:39:46
subtitle:
header-img: amazon.jpg
tags:
  - AWS
---

## Preface

這是一篇 2006 年 AWS CTO Werner Vogels 的 [interview](https://queue.acm.org/detail.cfm?id=1142065)，收錄在ACM queue 上面，過了將近 15 年後來看，令我特別有感觸，也非常佩服我偶像 Werner Vogels 的真知灼見。

我特別驚呀的是在 DevOps, Microservice 這些詞誕生前，就有了這則訪問，而這篇訪談又環繞在這幾點上面，談論不管在團隊或是技術架構上，Amazon 怎麼解決 scale independently 這個問題。

## Amazon store 成長帶來的問題

Amazon store 的成長帶來了很多技術上面的挑戰，面對不同的客戶和賣家，不同的 access Amazon services 的方式，Vogels 舉出了下列非常多需要考慮的問題，如何在 ultra-scale 下持續維持 availability 和 performance 變成迫切的需求。
> The impact has been on many areas: larger data sets, faster update rates, more requests, more services, tighter SLAs (service-level agreements), more failures, more latency challenges, more service interdependencies, more developers, more documentation, more programs, more servers, more networks, more data centers.

## Amazon 架構的轉變

Amazon 起初也是 monolithic application，所以一開始也都是針對系統瓶頸，也就是改善 backend 且 database 的部分，進而去支撐更多的 items, customers 和 orders，直到 2001 年 frontend 變成了瓶頸。很快的他們就發現 scale independently 這件事會被 sharing resources 所影響，沒有清楚的 isolation，沒有 ownership 減緩了開發速度。
>there were many complex pieces of software combined into a single system. It couldn’t evolve anymore. The parts that needed to scale independently were tied into sharing resources with other unknown code paths. There was no isolation and, as a result, no clear ownership.

經過了一陣子的 introspection，得出了 service-oriented architecture 是這些問題的解答，不單可以提供 the level of isolation，也可以讓 Amazon 的不同部門，可以快速且獨立的建構不同的 components。

service orientation 在 Amazon 的意義是，資料和 business logic 會被封裝(encapsulation)在一起，而且只能透過*公開的 service 介面*，不允許直接呼叫 database。Amazon 花了將近五年，把 two-tier monolith 的架構換成 fully-distributed, decentralized services。

每個 service 都有個別的 team 直接相關，這些 service 的死活都是這些 team 直接負責，讓 developers 去 operate services，不論從客戶的角度，或是技術的角度，都直接的提升了 service 的品質，這邊也直接呼應了 AWS 怎麼做 DevOps 的哲學。
>that team is completely responsible for the service—from scoping out the functionality, to architecting it, to building it, and operating it.

## AWS 技術的選型

訪者在後面也詢問 Vogels，如何看待像是 SOA, WSDL, SOAP, WS-security 等等的 Buzzword 。

而 Vogels 提到 AWS 在 SOA 這個詞還沒火起來前，就在改造內部的 services 了，這個時候內部主要用 WSDL/SOAP，不過有做很多的最佳化在 transport 和 marshalling 上面，不過對外就開始有提供 Rest-like 的介面，因為當時很多 php, perl 的 LAMP 架構的 library 被很多中小型客戶使用，而 SOAP 的介面主要提供給 Java 和 .NET 平台。在 Amazon 的角度來看，提供 Rest or SOAP 的介面選擇不是重點，重點則是客戶使用什麼，因為客戶只想要你提供最間單的 toolkit 給他們建構 application 。

## Amazon 商業上面的哲學

Amazon 非常重視客戶的 input，在構思新的產品的時候，一定會把客戶的 feedback 放進 loop 裡面。Amazon 如何 measure 產品是否成功，Vogels 博士提供了一個的觀點，是從客戶的角度出發，像是產品的更動是否有改變客戶的使用行為，像是有沒有減少幾個 steps 去找到他們要的 items，不過這些從人的 behavior 出發的 measure 也更難去偵測，而且要改變人的行為也很困難。
> We measure whether or not a new feature is successful in terms of customer satisfaction: Do people find things more easily? If we can improve the convenience of shopping on Amazon, then we have booked a major success.

## 談技術團隊和招人

Amazonians 每兩年也需要花時間去當一陣子 customer services，進而瞭解客戶的需求和想法，

Amazon 的招人哲學，就是看 candidate 怎麼思考 customer 和 technology。這邊提出 technology is useless if not used for the greater good of serving the customer。另外 working from customer backward 也是 A 社的重要哲學。

## 心得

這篇訪談其實讓我驚訝的是，在 SOA(2009)、DevOps (2009) 和 microservice(2015) 這些 buzzword 出現並且流行前，Amazon 早就嘗試在內部實現這種嶄新的方法，並且提煉出很多不錯的 practices，像是 service 的設計需要考量 sharing resources，怎麼樣將 service 的功能利用一致的介面 expose 出來，將 service 的 operation 責任丟給 service team 改善 ownership。另外是對於產品的思考角度，總是從客戶的方向開始出發，過去幾年，我在使用 Amazon 的服務的時候，其實也從這些 service 和服務的身上學到了不少， 能夠讓團隊更快速的開發產品，更快地迎合客戶的需求，真的是從流程面和技術面都需要值的提升，這個訪談讓我看到了 Vogels 如何看待 large scale 的產品和技術問題，很推薦大家去看原文。

## Reference
- [A Conversation with Werner Vogels](https://queue.acm.org/detail.cfm?id=1142065)

<span>Photo by <a href="https://unsplash.com/@bryanangelo?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Bryan Angelo</a> on <a href="https://unsplash.com/s/photos/amazon?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>
