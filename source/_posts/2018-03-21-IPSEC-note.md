title: IPSec 筆記
date: 2018-03-21 20:49:34
tags: AWS
---

# 前言 #

這篇筆記是用來記錄 IPSec protocal 的一些細節，前陣子在架設 AWS VPN 的時候，遇到了一些小問題，主要還是防火牆擋到需要走的 port ，而當時就在想自己對於 IPSec protocal 也太不熟悉了，所以才有這篇文章來稍微紀錄一下。

# 為什麼需要 IPSec #

IP is not secure，我想這點學過計算機網路的同學應該都會知道這點，而有可能受到以下的風險
- [source spoofing](https://en.wikipedia.org/wiki/IP_address_spoofing)
- [replay packets](https://en.wikipedia.org/wiki/Replay_attack)
- data integrity (資料受到竄改)

基本上使用 VPN 走 IPSec protocal 可以確保 CIA (似乎跟資安有關的都會提到這三個詞）
- Confidentiality: 利用演算法將資料加密 (DES, 3DES, AES & Blowfish)
- Integrity: 資料完整性，利用 hashing algorithm 保證資料沒有受到竄改
- Authentication: 認證

# IPSec security architecture #

- 使用 `Layer3 Network layer` 這層
- Application 層的大家可以無感的享受其 CIA 的好處
- Components
    - Authentication Header (AH)
    - Encapsulation Security Payload (ESP)
    - Security Associations (SA)

基本上我覺得要懂 IPSec，可以先來弄懂 AH & ESP 會比較重要，因為這兩個東西有對 IP packet 動手腳

## Authentication Header ##

AH 主要提供的是驗證 Data integrity & data origin source，然後沒有提供任何*加密*的功能，使用 HMAC 算法，把 payload & header 和 IKE 定義好的 key 一起拿來 hash，但這邊要小心因為 NAT 會改變 header，而被改變的話，另外一邊就沒辦法解析正確，所以基本上 AH 應該是不可能跟 NAT 共存。
而其中又分為 Transport mode & Tunnel mode，後面會有介紹有什麼不同。
AH 使用 port 51。

![AH.png](/img/2018-03/AH.png)

## Encapsulation Security Payload ##

ESP 的功能比起 AH 強大了許多，confidentiality, authentication, integrity 都包含在其中了，所以真正有提供加密的功能，而在驗證 Data integrity 方面，還是要看是使用 Transport mode 或是 Tunnel mode
- Transport mode: ESP 沒有對 IP header 做 hash ，所以只能保證 Data 是沒有被修改的
- Tunnel mode: 有將 IP header 包進來，所以這點跟 AH 是一致的

對照下圖可以發現，ESP 和 AH 最大的差別應該是 AH 會對於 Outer IP header 做驗證，所以其實 IPSec 唯有使用 ESP tunnel mode 才能和 NAT 共存。
而在 [RFC 3948](https://tools.ietf.org/html/rfc3948) 裡面也有寫道: `Because the protection of the outer IP addresses in IPsec AH is inherently incompatible with NAT, the IPsec AH was left out of the scope of this protocol specification.` 證實我們的推論應該是無誤的，難怪 AWS 的 NAT 教學裡面都是用 ESP 來做連線啊QQ。
ESP 使用 port 50。

![ESP.png](/img/2018-03/ESP.png)

## Transport mode & Tunnel mode ##

Transport mode: 通常是直接建立在兩台主機上，因為不需要再多加一個 IP header ，整體來說較省頻寬，在這個模式下，兩邊的主機都要安裝 IPSec 的 protocal，而且不能隱藏主機的 IP 位置。
Tunnel mode: 針對 Firewall 或是 Gateway proxy，一般來說我們會用這個模式，因為他們不是原本的發送收端。

## Security Associations ##

IPsec 中最重要的其實是 SA，因為它定義了如何協商，還有要使用哪些 Policy 和參數
1. Authentication method
2. Encryption algorithm & hashing algorithm
3. Life time of SA
4. Sequence number (避免 replay 攻擊)

而基本上 SA 是單向的，所以通常要建立兩條 SA (from A to B and B to A)，然後這些 parameter 會經過 Internet Key Exchange (IKE) protocal 來決定，IKE 主要有分兩個 step

IKE phase1: 主要做 Authenticate，Authentication 方面常常使用的都是 pre-shared key，基本上就是用同一組密碼，接著透過 Diffie-Hellman 來建立一組 Key，而這組 Key 是要被 Phase2 拿來用的。
IKE phase2: 處理 IPsec security 協商，最後 IPSec SA 完成，接下來才會建立 IPSec 的連線。

** IKE 主要走 port 500

## 結論 ##

這只是一篇小小的筆記，而網路上面有更多詳細的資料，但有了這些基本概念後，對於為什麼 VPN 打不通，會有更多除錯的方法，像是那些 port 是不是沒開，或是 SA 整個設定錯誤，導致雙方協商失敗等等，會讓我們更有方向。

# Reference

1. http://chunchaichang.blogspot.tw/2011/12/ipsec-nat-t.html
2. https://www.jannet.hk/zh-Hant/post/internet-protocol-security-ipsec/
3. https://en.wikipedia.org/wiki/IPsec#Security_association
4. https://tools.ietf.org/html/rfc3948
5. http://www.deepsh.it/networking/IPSec.html
