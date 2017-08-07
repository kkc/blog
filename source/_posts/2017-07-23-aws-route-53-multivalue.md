title: AWS route53 multivalue 筆記
date: 2017-07-23 14:02:36
tags: AWS
---

AWS 發表的新的 route53 routing policy: [multivalue](https://aws.amazon.com/about-aws/whats-new/2017/06/amazon-route-53-announces-support-for-multivalue-answers-in-response-to-dns-queries/)，Raddit 上面的某篇[文章](https://www.reddit.com/r/aws/comments/6iptol/amazon_route_53_announces_support_for_multivalue/)，有比較了一下跟 simple & weighted 的差別，但主要看這張圖就夠了 [http://imgur.com/4gvktlm](http://imgur.com/4gvktlm)。

1. simple 是把 multiple values (IP) 寫成一筆 record，但是 multivalue 是寫成多筆，然後用 nslookup 去查看起來回應的結果是差不多的。
2. 但其實還是有一點不同，[文件](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html) 上面寫到 multivalue 最多只會 randomly 回 8 筆 healthy records。
3. multivalue 的好處是，每筆記錄可以跟 health check associate 在一起，當 endpoint failure 時會自動把這個 IP 拿掉。
4. weighted 的話，雖然也是多筆 records，但是用 nslookup 去查，基本上只會回一筆資料給你。
5. multivalue 如果全部 endpoint 都不 healthy，他最多也會回 8 筆 unhealthy records 。

有了 multivalue 後其實原本只能用 weighted 做的事情就可以移過來了，像是用 ASG 開一堆機器，但是想用 DNS RR 來做 load distribution 就可以改用 multivalue 來實作，debug 時也可以看到全部的 dn list，對我來說更有彈性一點。
