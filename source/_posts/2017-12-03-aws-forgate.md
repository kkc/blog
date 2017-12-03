title: AWS forgate 的簡短筆記
date: 2017-12-03 16:32:48
tags: AWS
---

AWS reinvent 2017 推出了許多對 container 管理的新工具，基本上我原本以為有 K8s on AWS 就很厲害了，沒想到 AWS 也沒有想放棄 ECS，在這兩個 container orchestration 的方法上面又疊加了一層 managed system -- forgate。

這篇主要是對這個 youtube 做的簡短筆記，但這裡他是用 ECS 作為例子，而針對 EKS 的 forgate 應該要等到 2018 才看得到了。

{% youtube 0SceSgOTyrw %}

# Forgate #

- No instance management
- Task native API
- Resource based pricing

有談到 per seconds billing，根據選擇的 resource 類型來計價，會是新的 pricing model

## 改良

以前在使用 ECS 的時候，其實有很多地方還是需要管理

- ECS cluster
- EC2 instances (HA, resource provision, mantainence)
    - docker daemon
    - ECS agent
    - OS version
    - ...etc

而在使用 forgate 後，我們可以完全不去管理 EC2 這層，感覺概念跟 serverless 很像，我就把 task 丟上去，請你幫我好好管理後面調度的部分。

## Task definition

forgate w/ ECS 也是沿用 ECS 的 task definition

Task definition
   - Task size
   - Task execution role
   - network configuration for task placement
       - VPC
       - Subnet
       - Security Group

## Network w/ forgate

[!network](/img/2017-12/forgate_network.png)
針對每個 task 會有不同的 ENI 可以使用

- AWS VPC networking mode - each tasks get its own ENI
- Full control of network interface vis SG and NACLs
- Public IP support

## security

權限劃分很重要

- cluster level isolation
- permission
  - cluster permission
      - Who can run/see tasks in the cluster
  - Application permission
      - Which of mu AWS resource can this task access
  - House keeping permission
      - ECR
      - cloudwatch
      - ...etc

# 結語

整個看起來，使用 forgate 好像會讓 container 管理變得更方便，但穩定性也是一大考驗，像是 AWS 的 ES，因為碰不到底層的 EC2，改個設定常常要跑半天，想起來如果用更 high level 的管理方式，常常改動需要等待更久的時間，也是犧牲了一點彈性，希望 forgate 不會有給我這樣的感覺。另外一方面，其實也有想過 AWS 的權限管理部分，真的需要有另外一層東西包起來，才能夠好好處理，這點我覺得 AWS 真的花蠻大的心力去解這問題，要不然直接用 k8s 跑，似乎在 compliance 上面都會有不少的難題要解。

Forgate 目前也只有少部分 region 可以使用，這邊許個願能夠在明年年中之前，讓大部分的 region 可用，這樣應該就夠了吧 (誤
