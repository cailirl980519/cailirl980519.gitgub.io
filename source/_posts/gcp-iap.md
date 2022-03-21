---
title: 透過Cloud IAP遠端連線至Linux VM
date: 2022-03-21 10:45:22
cover: https://mrtang.tw/wp-content/uploads/2017/09/1505355061-24b1fd8e4fa17ca39961de99073f397a.png
categories: GCP
tags:
- IAM
- IAP
- Compute Engine
- GCP
- Linux
---

講述Cloud IAP前，先來說一下Cloud IAM，Cloud IAM的全名為**Identity and Access Management**，顧名思義就是身份與權限管理服務，管理員可以透過設定IAM，來指派使用者的身份，並授權使用者存取GCP內的資源。

而Cloud IAP(Identity-Aware Proxy)中的TCP-Forwarding則是透過HTTPS的加密安全連線和IAM的權限管理來建立快速的安全連線，讓我們能夠在不使用外部IP的情況下，也能輕鬆又安全的連接上我們的Compute Engine。

![](https://storage.googleapis.com/cloudace-tw-blog/1/2021/11/IAP_TCP_Forwarding-blog-images-12.png)

*取自[Cloud Ace](https://blog.cloud-ace.tw/identity-security/iap-tcp-forwarding/)*

如圖所示，透過IAP TCP-Forwarding，只需將VM的port通道打開，就可以直接透過通道連接至VM。

|           |  Linux   | Windows         |
|  :----:   | :----:   | :----:          |
|  連接方式  |  SSH     | RDP(遠端桌面連線) |
|  預設port |   22     |      3389       |

今天就來紀錄一下如何使用Cloud IAP建立Linux VM連線，並使用GCP Web SSH來連線至VM

## 設置防火墻
首先先新增一個用於Cloud IAP連線的防火牆規則
打開**虛擬私有網路(VPC network)** > **防火牆(Firewall)** > **建立防火牆規則(Create VPC Network)**
![Imgur](https://i.imgur.com/PXb3ESC.png)

接著填入以下配置並建立:
- 名稱：`allow-from-iap`
- 流量方向：輸入
- 目標：網路中所有執行個體
- 來源篩選器：IPv4範圍
- 來源IPv4範圍：`35.235.240.0/20`
- 協議及端口：`TCP`:`22`(允許SSH)
![Imgur](https://i.imgur.com/pcyMbVE.png)

:::tip
防火牆預設會有`default-allow-ssh`及`default-allow-rdp`這兩個規則，這兩個規則會允許所有主機連線至VM，建議可以停用或刪除
:::

## 設置IAM權限
開啟 **IAM與管理** > **身分與存取權管理** > **新增**
- 新增主體：需要連線的email
- 角色(Role)：
  + Compute執行個體管理員 / Compute Instance Admin（v1）
  + 受IAP保護的通道使用者 / IAP-secured Tunnel User
    新增以下條件:
    + 名稱： `limit port`
    + 條件編輯器：`destination.port == 22`
  + Service Management管理員 / Service Management Administrator

![Imgur](https://i.imgur.com/k4kzShW.png)

接著開啟**IAM與管理** > **服務帳戶** > 勾選**Compute Engine default service account** > **管理存取權** > **新增主體**
![Imgur](https://i.imgur.com/pPz7FdB.png)
- 主體：需要連線的email
- 角色：編輯者

## 啟用IAP
進入**安全性** > **Identity-Aware Proxy** > **SSH和TCP資源**，若未啟用API請先啟用，啟用完成則會看到以下畫面
![Imgur](https://i.imgur.com/VY0a846.png)

## GCP Web SSH登入測試
最後登入主體帳戶，進入Compute Engine點擊SSH連線
![Imgur](https://i.imgur.com/F4KSBtE.png)

若看到以下畫面就大功告成啦～
![Imgur](https://i.imgur.com/MMqBE0p.png)

## 參考資料
- [比 VPN 連線、跳板機更方便！利用 Cloud IAP 快速建立遠端安全連線 - Cloud Ace 技術部落客](https://blog.cloud-ace.tw/identity-security/iap-tcp-forwarding/)
- [什麼是 Cloud IAM - Cloud Ace 技術部落客](https://blog.cloud-ace.tw/identity-security/what-is-cloud-iam/)
- [Using IAP for TCP forwarding | Identity-Aware Proxy | Google Cloud](https://cloud.google.com/iap/docs/using-tcp-forwarding)