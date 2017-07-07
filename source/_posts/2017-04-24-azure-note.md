title: Azure VMSS 的簡略使用筆記
date: 2017-04-24 22:25:18
tags: Azure
---

在踏入 Azure 領域前，因為原本比較熟悉 AWS，所以從 AWS 的 Service 尋找對應的 Azure Service 算是比較好入門的管道，基本上這個連結 [Azure to AWS mapping](https://docs.microsoft.com/zh-tw/azure/architecture/aws-professional/services) 算是整理的很不錯，不過也可以看得出來 Azure 也是設定大家都懂 AWS，想要從這邊搶客戶過來。

而會想試用 Azure，主要是 Azure 有新的 M60 的 GPU instance 可以玩，相比起 AWS 的 K80 好像強大不少，所以家裡老大叫我去測試一下。 接下來的幾篇文章主要會紀錄怎麼使用 Azure cli 到最後部署 GPU instance。

---

# Azure CLI

在被 Azure Portal 荼毒一陣子後，決心轉向使用 Azure 的 CLI，在這邊也要向大家推薦，Azure 的 CLI 真的比 Portal 好用一萬倍，很多東西都是 CLI 有但是 Portal 還在生的階段。

## Azure CLI Docker

如果覺得安裝很麻煩，這邊也有 Docker image 可以直接抓下來用

- [https://github.com/Azure/azure-cli-docker](https://github.com/Azure/azure-cli-docker)
- [https://hub.docker.com/r/azuresdk/azure-cli-python/](https://hub.docker.com/r/azuresdk/azure-cli-python/) -> Microsoft Azure CLI 2.0 - Preview

使用 `az login` 後就會產生相關的 config file，如果使用 docker container 想要保存其 config 的話，記得 mount 一個外部的 volume 進去。


# Azure resource group 簡介

基本上現在使用 Azure，都會推薦你使用 Resource Group 來管理你的 Infrastructure ，這邊感覺跟 AWS 的 tag 有點像，就是可以把相關的組件分到同一個 Group 下面，如果想要看詳細的介紹可以左轉[這邊](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview)找

而我主要就是記錄一下怎麼用 Azure CLI 開個 VM & resource group 。

## 建立 ARM & VM

![Azure Resource Group](https://image.slidesharecdn.com/2016-02-25-vmss-ug-deep-dive-160226160609/95/uk-azure-users-group-7-638.jpg?cb=1456502858)
Azure resource group 的階層結構

- Azure Resource Group (把相關的組件加個這個 Group 下面管理）
  - VNET
    - Subnet
      - NIC

而 VM 需要有 NIC 去得到 private ip/public ip

1. 建立 Resource Group

    ```
    azure group create myTestGroup -l southcentralus
    ```

2. 建立 Storage Account

   這裡跟 AWS 有很大的差別，AWS 的 EBS 是獨立的架構，而 Azure 的 Storage 在使用前都需要建立 Storage Account，而底下又有四種類別 Blob, File, Tables, Queue，Azure VM 的 OSdisk & DataDisk 都是存在 Blob 裡面，然後建立 Custom Image 也會被建立在同個 Storage Account 裡面。

    ```
    azure storage account create -g GROUPNAME \
        -l LOCATION --sku-name LRS --kind storage STORAGENAME
    ```

3. Create virtual net

    ```
    azure network vnet create myTestGroup TestVnet -l southcentralus
    ```

4. Create subnet

    ```
    azure network vnet subnet create -a 10.0.0.0/16 myTestGroup TestVnet TestVnetSubnet
    ```

5. Create public IP

	```
    azure network public-ip create myTestGroup myTestPulibIP -l southcentralus
    ```

6. Create Network interface (NIC)

	```
    azure network nic create myTestGroup myTestNic -k TestVnetSubnet -m TestVnet -p myTestPulibIP -l southcentralus
    ```

7. Create VM

	```
    azure vm create -n myTest -l southcentralus -g myTestGroup -f myTestNic -z Standard_NV6 -y Linux --storage-account-name kakashistorage --generate-ssh-keys -Q Canonical:UbuntuServer:16.04.0-LTS:latest
    ```

## 建立 Custom Image

在會開 VM 後，想要學習怎麼建立自己的 Custom Image，有點像是在 AWS 裡面去打 VM 的 AMI，這也是建立 immutable image 的一環。
參考資料為[這篇](https://docs.microsoft.com/en-us/azure/virtual-machines/linux/capture-image)

1. Deprovision your source VM

    ```
    ssh ops@myvm.westus.cloudapp.azure.com
    sudo waagent -deprovision+user -force
    exit
    ```

2. Deallocate the VM with az vm deallocate

    ```
    az vm deallocate --resource-group myResourceGroup --name myVM
    ```

3. Generalize the VM with az vm generalize

    ```
    az vm generalize --resource-group myResourceGroup --name myVM
    ```

4. Create an image from the VM resource with az image create:

    ```
    az image create --resource-group myResourceGroup --name myImage --source myVM
    ```

下完建立 image 的指令後，其實會在相對應的 Azure blob Storage 下面開一個專門的資料夾去存這個 snapshot，然後檔名是 vhd
`e.g: https://blahblahblah.blob.core.windows.net/images/Microsoft.Compute/Images/vhds/MyImage-osDisk.ef68b9e6-8081-4082-972c-2375b39953f8.vhd`

感想: 這裡其實覺得比 AWS 難用多了，還需要做那麼多步驟。

(Optional) Create a VM from your image resource with az vm create

    ```
    az vm create --resource-group myResourceGroup --name myVMDeployed --image https://blahblahblah.blob.core.windows.net/images/Microsoft.Compute/Images/vhds/MyImage-osDisk.ef68b9e6-8081-4082-972c-2375b39953f8.vhd --admin-username azureuser --ssh-key-value ~/.ssh/id_rsa.pub
    ```

## 使用 Template 部署 VMSS

所謂的 VMSS 其實就是 Azure 版本的 Auto scaling group，基本上 MS 推薦大家在 Azure 上面用 Template 去作部署 (有點像 AWS 的 cloudformation) ，而很棒的地方是，他們在 Github 上面擺了一個 [azure-quickstart-template](https://github.com/Azure/azure-quickstart-templates) 非常實用，可以拿來一鍵部署，或是修改成你想要的架構。

而我則是使用以下這位大大的 template 去做 VMSS + custom image 的部署

[https://github.com/gbowerman/azure-myriad](https://github.com/gbowerman/azure-myriad)

這裡他提供了一個 vmss-linux-customimage.json，點選 deploy to Azure 會發現需要填入一些選項，像是 VM sku 還有 Image 的位置，這邊只要填入剛剛打好的 Custom Image 的位置，就可以作簡單的部署了，而實務上面其實會把改完的 template 利用 Azure CLI 去執行 deploy 。

不過有一件事情要注意，如果要在其他 region 開這個 custom image，需要將他 copy 到同樣 region 下的 Azure Storage，這點跟 AWS 一樣。

## 使用 Load Balancer

當使用的流量上升時，我們會需要使用 Load Balancer 來分流到多個 service，達成 Horizontal Scaling 的需求，用上面提供的 template 也會自動產生 LB 擋在前面，而 VMSS 妙的地方在於需要設定 NAT rule 才能連入裡面的每台 VM，可以參考 Erics 大大的[文章](https://www.azure-vm.recipes/ch04/dispatch-requests-to-multiple-vms.html) 了解一下怎麼設定，接著就可以用 ssh <ip> -p port 去連線，這個跟 AWS 有很大的差別。

### Health Check

跟 AWS 一樣需要設定 Health Check 去確定 Backend service 是不是活著，不過 Portal 介面上面看不到，LB 怎麼判斷後面連到的 VMSS 哪一個 instance 是 Healthy or Unhealthy，這點真的有點讓人覺得討厭。

### LB 的議題

- 沒辦法知道 Instance Healthy or Unhealthy
- 沒辦法更動 Health Check 的 Rule
- 沒辦法確定是否有 Connection Draining
- 沒辦法設定 Timeout
- 能看的 Metrics 太少，或是我在 Portal 中不知道要去哪裡找 (e.g request count, response time, response status)
- 不知道怎麼隨便加普通 VM 在某個 LB backend
- 不知道 scale in 時能不能 protect 某個 instance 不受影響

# 其他

## 好用的 debug 工具

這個 tool [Azure Resoure Explorer](https://resources.azure.com/) 非常好用，可以讓你直接檢查每個 component 的屬性，而不用一直用 CLI query 其中的狀態，而且有些狀態在 Portal 上面非常難以尋找。

## VMSS 特點小記

根據這篇文章記錄一下 [http://www.babylon365.net/got-vm-scale-sets-aka-msazurevmss/](http://www.babylon365.net/got-vm-scale-sets-aka-msazurevmss/)

- 可以有 Scale in & Scale out 的功能，不過對比 AWS 好像也不是很容易在 Portal 上面設定？
- 一個 VMSS 只能有一個 Public IP
- 可以設定一個 extension，然後全部的 VM 都 applied 同一個
- 支援 Rolling Upgrade
- 不 Support Data Disks
- Auto scaling 目前只吃 CPU Metrics?
- 沒辦法讓其中的 VM 有 Public IP
- 不支援 Disk encryption

## 抱怨

- VM Extension 執行完後，常常 VM 的狀態還是在 Updating 而不是 Complete，最久等過兩個鐘頭才完成，但是 script 早就跑完了...(汗。
- Scale out 機器後，想要 Scale in ，居然沒辦法選擇用什麼 Policy 關機器，而且總是關最新打開的，最後 Rolling upgrade 只好人工寫關閉舊的 VM。
- Portal 少 CLI 太多功能，感覺變得很不實用。
