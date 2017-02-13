title: ubuntu16.04-docker-aufs
date: 2016-08-22 08:28:23
tags: docker
---

## Background ##

事情是這樣的，前陣子在 AWS 上面使用最新的 [ubuntu image](https://cloud-images.ubuntu.com/locator/ec2/) 來建置 production 的環境，大家都知道 ubuntu 16.04 已經採用~~萬惡~~的 systemd 來管理 ubuntu 上面的 process，而在我開開心心使用 systemd 來管理我的 docker 時，就這麼發生了意外，每當我打好 AMI 再使用這個 AMI 開機時，就會發現 docker 掛了，而且時好時壞，讓我全無頭緒。

## 追查 ##

再重開機後看了 `/var/log/syslog`，裡面馬上發現了下列這個錯誤

```
ERRO[0000] [graphdriver] prior storage driver "aufs" failed: driver not supported
FATA[0000] Error starting daemon: error initializing graphdriver: driver not supported
```

原來是 aufs 沒有掛上去，不過奇怪的是，我明明已經照著 docker [文件](https://docs.docker.com/engine/installation/linux/ubuntulinux/#/prerequisites-by-ubuntu-version) 上面安裝好了，而且在開機前也 run 得很開心，究竟是為什麼呢？

備註: Aufs 沒有被包在原生的 kernel 裡面，所以按照文件要使用 sudo apt-get install linux-image-extra-`uname -r` 安裝

## 原因 ##

在追查了一陣子後才知道，原來是因為裝好後，還要使用 modprobe aufs ，這樣之後重開機才會自動載入啊，docker 文件好像好像沒看到這點，而詳細能解決這個問題的答案，在看懂了這兩篇後才懂：

   * https://meta.discourse.org/t/discourse-docker-installation-without-aufs/15639/12
   * http://stackoverflow.com/questions/33357824/prior-storage-driver-aufs-failed-driver-not-supported-error-starting-daemon

解決的方法就是用
```
sudo apt-get install linux-image-extra-`uname -r` && sudo modprobe aufs
```
之後要升級食用
```
sudo apt-get update && sudo apt-get upgrade && apt-get -y install linux-image-extra-$(uname -r) aufs-tools
```

## 感想 ##

這個坑在 ubuntu 上面真是很煩，希望 docker 文件趕快補上啊啊啊，另外如果不想用 aufs ，其實也可以用 overlayfs，這個 kernel 就有直接支援，所以就不用搞那麼多奇奇怪怪的東西了。
