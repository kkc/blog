title: 記一次 AWS EKS troubleshooting 的歷程
description: AWS 的 Kubernetes 蠻多東西要學的啊（抓頭)
date: 2018-09-11 15:33:58
catalog: true
tags:
  - AWS
  - Kubernetes
  - EKS
catagories:
  - Kubernetes
header-img: "EKS-bg.jpeg"


---

# 前言
千呼萬喚下，前陣子 AWS 版本的 Kubernetes 終於 [GA](https://aws.amazon.com/blogs/aws/amazon-eks-now-generally-available/) 了，但其實真正玩過後，跟之前用 CoreOS 跑起來的 Kubernetes 有不少的差別，主要是因為 AWS 需要把 IAM 及 Network 部分和 Kubernetes 做整合，因為權限和網路效能其實對於服務都是很重要的，而我也是在跑過 EKS 和經歷了這個除錯的過程，才更讓我了解到在不同平台上跑 Kubernetes 的差別，尤其要把 AWS IAM & RBAC 怎麼連接搞懂，還有 packet 在 AWS & k8s network 中怎麼流，都是很重要的，所以可想而知，在 AWS 上面疊一層 Kubernetes ，其實對 operation 來說，整體複雜度是上升的。

# 問題
一開始把玩 EKS 是照著 Pahud 大的 [workshop](https://github.com/pahud/amazon-eks-workshop) step by step 跑起來，而使用 eksctl 其實是很舒服的，可以直接幫你把 worker node 架好，並且把 autoscaling group 和 cloudformation 都設定好，整體來說簡化了很多步驟，所以體驗還不錯，但是公司在管理 infrastructure as code 的部分，比較希望一致使用 terraform 而不是 cloudformation。 我的問題就發生在當我用同事改寫的 terraform template 啟 EKS 時 ，一直沒辦法用 kubectl get nodes 得到 node，另外就是起好的 node (AWS instance) 上面也一直沒看到 secondary ip 出現，這跟我一開始使用 eksctl 看到的結果有落差。

# 除錯
因為一直無法讓 worker node 加進去 k8s cluster，感覺是個不錯的機會，可以學習 k8s 是怎麼運作的，我想到的第一步是登入機器使用 `journalctl -u kubelet` 觀看 kubelet 是否有什麼有趣的 log ，立馬就發現有 `Unauthorized` 的 Error，後來查看 [EKS User Guide Troubleshooting](https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html) 的 **Worker Nodes Fail to Join Cluster** 的章節 ，發現少做了 `kubectl apply -f aws-auth-cm.yaml` 這步，因為 AWS 的 node 要加進 cluster 內，也是需要把 Authorization 這塊設定起來，要不然 master 不會無緣無故的讓你呼叫，而其實 ekectl 有幫忙處理這塊，但使用 terraform deploy 時，就需要手動去執行這步，才能讓這個 nodes 註冊到 cluster 上面，而這之後也是我們的 script 可以改進的部分。

1. Download the configuration map:

    ```
    curl -O https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-08-30/aws-auth-cm.yaml
    ```
2. Open the file with your favorite text editor. Replace the `<ARN of instance role (not instance profile)>` snippet with the NodeInstanceRole value that you recorded in the previous procedure, and save the file.
   這邊要把 rolearn 這邊填成正確的 ec2 instance role 的 ARN，如果填錯了就可能一直連不上喔

    ```
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapRoles: |
        - rolearn: <ARN of instance role (not instance profile)>
          username: system:node:{{EC2PrivateDNSName}}
          groups:
            - system:bootstrappers
            - system:nodes
    ```

3. Apply the configuration. This command may take a few minutes to finish.

    ```
    kubectl apply -f aws-auth-cm.yaml
    ```

在加完 worker node 後，還是發現 pod 還是無法開始成功得到 IP，所有的 container 都處在 `containerCreating` 的狀態中，這時就有點懷疑是不是 `AWS-VPC-CNI` 的問題，因為在 EKS 架構中，每一個 pod 都會被分配到一個 IP，CNI 基本上就在負責這件事，接著在後續的搜索中找到了 AWS 在 CNI GitHub Repo 上面放了一個很不錯的 [Troubleshooting note](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/troubleshooting.md)，透過這個 note，必須要檢查下列事項，就可以慢慢釐清問題是什麼。

1. 對應的 subnet 中能使用的 IP 數量是不是足夠
    * 這邊也提到有可能有 Leaked ENI 的問題，就是 EC2 關閉後但是沒有把 ENI 砍掉，之前被 attach 到這張 ENI 上面的 secondary ip 就不會被釋放出來。
2. 確認是不是該 instance 能使用的 ENIs & IPs 是否足夠

基本上就是要看 ENI 能不能分配到 IP，還有該 instance 上面的 pod 數量是不是大於分配到的 IP 數量

Debug 的流程

1. Check `/var/log/aws-routed-eni`
   如果能看到
   ```
   2018-09-11T07:28:11Z [DEBUG] Start increasing IP Pool size`
   2018-09-11T07:28:11Z [DEBUG] Skipping increase IPPOOL due to max ENI already attached to the instance : 3
   2018-09-11T07:28:16Z [DEBUG] IP pool stats: total = 15, used = 9, c.currentMaxAddrsPerENI = 6, c.maxAddrsPerENI = 6
   ```
   就代表沒什麼問題

2. 而我在確認 log 後發現這行 `[ERROR] Failed to get eni limit due to unknown instance type t3.medium`，後續也查到了這張 ticket [https://github.com/aws/amazon-vpc-cni-k8s/pull/145](https://github.com/aws/amazon-vpc-cni-k8s/pull/145)，還需要等到下個版本 release 後才能使用 t3.medium 的 instance type.

3. 在把 instance type 改回 t2.medium 後，的確整個網路就通了，pod 也可以成功跑起來。

# 後記

各家 cloud provider 都有提供 Managed k8s 的 service，除了 GKE 外，其他家 provider 為了讓 k8s 跟自己的系統整合的更好，都多少加了一些東西上去，像是 AWS 為了整合其 IAM 和 RBAC，還有開發 CNI 整合 VPC networking 去改善原生 Flannel 的效能問題，新增加上的東西往往也變成我們的知識負擔，不過 EKS 其實也是讓我們省下很多問題，像是 k8s 版本要升級，還有 master node & etcd 的管理問題，都丟給 AWS 去管理，我們不用頭痛這塊，個人覺得 Kubernetes 最需要的就是 master node 要穩定，Managed k8s service 可以幫我們很好的解決這些問題，但是我們還是要懂原本 AWS 的架構，像是 IAM, security group, VPC subnet，也要了解 k8s node 或是 pod 有問題時，要從哪個方向去找解法，像是 pod & pod 之間的 networking，或是 RBAC 是不是設定有誤，導致讀不到需要的東西，就算有了 EKS，還是得去了解這些底層的東西才會讓你管起來更有信心，接下來有機會會繼續寫一下 EKS 的一些筆記。


# Reference
1. https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt 不同的 instance family 能用的 pod 數量上限 (跟能 attach 的 ENI & IP 有關)
2. https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI
3. https://github.com/awslabs/amazon-eks-ami/issues/27
4. https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/troubleshooting.md
