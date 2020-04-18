---
title: AWS SSM session manager 筆記
catalog: true
date: 2020-04-11 21:09:31
subtitle:
header-img: header.jpg
tags:
  - AWS
---

一直以來，如何登入到 AWS EC2 instance 就是個大問題，以往的方式都是在建立 Instance 的時候，設定其 [key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html) ，再把 private key 好好的保存下來，不過這個方式對於管理許多機器的人，其實是很煩人的，試問有多少人會乖乖的 rotate 機器上面的 key，而在有很多服務和機器的情況下，對於這些 key 的生命週期管理是非常重要的。

再者，在有些情況下，還是會見到把 EC2 instance 變成所謂的寵物機，然後見到一堆人的 key 被加入到 `.ssh/authorized_keys ` 內，以便大家可以登入存取，這種方式讓我們更難的去顧到機器安全，在人員離開後，也不知道有沒有正確的把那些 key 拔掉。

通常來說，為了安全性，我們都會建立所謂的 Bastion host (跳板機) 還有 ip whitelist 及 VPN，去規範讓有權限的人去存取機器，不過不管是哪種方式也好，其實都增加了管理上的成本。

AWS 推出了 session manager 很好的幫我們解決了這個問題，而去年也推出了一些新服務，可以讓我們 [scp](https://aws.amazon.com/about-aws/whats-new/2019/07/session-manager-launches-tunneling-support-for-ssh-and-scp/) EC2 上面的檔案或是利用 [portforwarding](https://aws.amazon.com/blogs/aws/new-port-forwarding-using-aws-system-manager-sessions-manager/) 的方式，讓我們從 local 機器測試 private VPC 內的服務，這篇筆記會列出該如何使用 session manager 以及相關的 IAM 設定。


## 安裝

### Local machine 需求

遵照[官方文件](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)
1. 安裝最新版的 aws cli，版本需要大於等於 `1.16.12` 才能使用
2. 安裝 [session manger plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

### EC2 需求

預設 session manager 是沒有權限可以碰 EC2 的，需要修改 instance profile 和加裝 ssm agent。 
1.  Create an IAM instance profile for Systems Manager (https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-profile.html)
    - 需要有 `AmazonSSMManagedInstanceCore` 
    - 另外可以參照 [minimul s3 bucket permission](https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent-minimum-s3-permissions.html) 
      ```
       {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::aws-ssm-us-east-1/*",
                "arn:aws:s3:::aws-windows-downloads-us-east-1/*",
                "arn:aws:s3:::amazon-ssm-us-east-1/*",
                "arn:aws:s3:::amazon-ssm-packages-us-east-1/*",
                "arn:aws:s3:::us-east-1-birdwatcher-prod/*",
                "arn:aws:s3:::patch-baseline-snapshot-us-east-1/*"
            ]
        }
      ```
2. 確認 instance 上面都有安裝好 [SSM agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-manual-agent-install.html)，AWS 上面新版的 ubuntu & amazon linux2 都有先裝好了，不過舊的 AMI 就需要自己去安裝。  

## 使用者設定

根據[文件](https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-restrict-access-quickstart.html)，設定 user 對應的 iam policy 權限，以下是個簡單的範例

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:StartSession"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:instance/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeSessions",
                "ssm:GetConnectionStatus",
                "ssm:DescribeInstanceProperties",
                "ec2:DescribeInstances"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:TerminateSession"
            ],
            "Resource": [
                "arn:aws:ssm:*:*:session/${aws:username}-*"
            ]
        }
    ]
}
```

進階一點，我們可以使用 tag 去區別用戶能夠存取的環境，像是 staging or production

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:StartSession"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:instance/*"
            ],
            "Condition": {
                "StringLike": {
                    "ssm:resourceTag/Environment": [
                        "staging"
                    ]
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:TerminateSession"
            ],
            "Resource": [
                "arn:aws:ssm:*:*:session/${aws:username}-*"
            ]
        }
    ]
}
```

設定完以上的基本設定後，就可以透過下列的指令去登入機器

```
aws ssm start-session --target i-0b0d92751733d1234
```

## 使用 scp

### 設定
這邊筆記下如何透過 session manager 去達成 scp ，基本上透過 AWS 文件  [session-manager-getting-started-enable-ssh-connections](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html) 上的描述，可以得知是利用 Proxycommand 透過 AWS tunnel 直接連接到我們的 EC2 機器上。

編輯 `~/.ssh/config` 並加入
```
# SSH over Session Manager
host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
```

就可以使用

```
scp -i -i /path/my-key-pair.pem test123 ubuntu@i-0b0d92751733d1234:~/test123
```

注意這邊還是要利用一開始設定好的 key pair 去做連線。

### 進階設定

上面提供的方法雖然可以讓我們使用 scp & ssh，但是有點討厭的是還是得設定 EC2 機器的 key，那有沒有辦法繞過去呢? 答案是有的，只是需要透過一個比較 tricky 的方式。

網路上有人寫好了這個 proxy command 的 [script](https://gist.github.com/qoomon/fcf2c85194c55aee34b78ddcaa9e83a1)，使用的方式很簡單

1. 下載並且把這個 script 放到 `~/.ssh/aws-ssm-ec2-proxy-command.sh`
2. 修改 aws-ssm-ec2-proxy-command.sh 成為可以執行
3. 修改 `~/.ssh/config` 裡面的指令

```
host i-* mi-*
  ProxyCommand ~/.ssh/aws-ssm-ec2-proxy-command.sh %h %r %p
```

就不用在帶一把 key 去做認證了

```
scp test123 ubuntu@i-0b0d92751733d1234:~/test123
```

其實原理很簡單，利用 `aws ec2-instance-connect send-ssh-public-key` 去建立一個 short-lived 的 key，這個指令詳細的好處可以看這篇 aws 文章 [new-using-amazon-ec2-instance-connect-for-ssh-access-to-your-ec2-instances](
https://aws.amazon.com/blogs/compute/new-using-amazon-ec2-instance-connect-for-ssh-access-to-your-ec2-instances/)，接著再使用這把 key 透過原本的 start session 那條路連上遠端的 ec2 機器。

## Port forwarding

這邊要再提供一個很有趣的方法，可以讓人透過 port forwarding 去連接 EC2 上面的服務，很多時候我們會把服務都放進 private subnet 內， 而 developer 想要測試這些 services 時，往往要利用 VPN 或是開一台在內網的 EC2 去連結，而使用 port forwarding 可以讓我們更容易地達成這個需求。

```
aws ssm start-session --target i-0b0d92751733d1234 --document-name AWS-StartPortForwardingSession --parameters '{"portNumber":["80"],"localPortNumber":["9999"]}'
```

這樣就可以透過 localhost:9999 去連結到 EC2 上面 service 的 80 port 了，詳細的內容也可以看這篇 AWS 的文章 [new-port-forwarding-using-aws-system-manager-sessions-manager](https://aws.amazon.com/blogs/aws/new-port-forwarding-using-aws-system-manager-sessions-manager/)

## Takeaway

- 使用 session manger 可以減少 key 的管理，減少資安漏洞
- 透過 proxycommand 可以讓我們建立 ssh tunnel，進而可以使用 scp 等等工具
- port forwarding 可以幫助 developer 測試在 private subnet 的服務
- 搭配 aws cliv2 可以透過 SSO 增加系統安全


## Reference
- https://www.youtube.com/watch?v=nzjTIjFLiow
- https://www.youtube.com/watch?v=kj9NgFfUIHQ
- https://aws.amazon.com/blogs/compute/new-using-amazon-ec2-instance-connect-for-ssh-access-to-your-ec2-instances/
- https://aws.amazon.com/blogs/aws/new-port-forwarding-using-aws-system-manager-sessions-manager/
- https://globaldatanet.com/blog/ssh-and-scp-with-aws-ssm
