---
title: Linux硬碟的大小事
date: 2022-03-04 09:07:52
cover: https://5.imimg.com/data5/BB/KJ/MY-8593783/laptop-hard-disk-500x500.jpg
categories: Linux
tags:
- Linux
- Debian
- GCP
---

今天想要來幫目前已經在運行的GCP VM來掛載一個新的硬碟，使會持續變多的資料不要存在開機碟，而是把他掛載到新的硬碟裡，這樣之後也可以方便地隨著資料增加而擴增。

## 新增磁碟
### GCP端
首先要從GCP選擇需要增加硬碟的VM，並點擊編輯來新增一個新的硬碟或已存在的硬碟，選擇完後按儲存GCP這邊就新增完成了！
![Imgur](https://imgur.com/mgnuN28.png)

### Linux端
> 以下指令皆使用root呼叫

#### 確認磁碟狀態

第一步先列出目前有的硬碟，而我新增的硬碟叫**sdb**，大小為256GB
``` console
$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0   10G  0 disk 
├─sda1    8:1    0  9.9G  0 part /
├─sda14   8:14   0    3M  0 part 
└─sda15   8:15   0  124M  0 part /boot/efi
sdb       8:16   0  256G  0 disk
```

再來確認一下硬碟的掛載狀態，可以看到目前我們新增的硬碟還沒有被掛載
``` console
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            2.0G     0  2.0G   0% /dev
/dev/sda1       9.7G  1.9G  7.4G  20% /
/dev/sda15      124M  5.7M  119M   5% /boot/efi
```

#### 格式化磁碟
接下來我們使用`mkfs`來格式化硬碟，並使用`ext4`文件系統，之後便可以快速增加硬碟大小，不需修改分區
``` console
$ mkfs.ext4 -m 0 -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
```
> -m 0: 使用所有可用的磁碟空間
> -E  : 最大限度的提高磁碟性能

#### 掛載磁碟
接著創建一個掛載磁碟的目錄，這邊創建一個`/data`目錄
``` console
$ mkdir -p data
$ ls
bin   data  etc   lib    lib64   lost+found  mnt  proc  run   srv  tmp  var
boot  dev   home  lib32  libx32  media       opt  root  sbin  sys  usr
```

再來使用`mount`來將磁碟掛載到對應的目錄
``` console
$ mount -o discard,defaults /dev/sdb /data
```

並讓所有用戶有讀寫權限
``` console
$ chmod a+w /data
```

#### 設置開機時自動掛載
獲取磁碟的**UUID**
``` console
$ blkid dev/sdb -s UUID -o value
ccff9e1e-b518-4ac5-aabd-e37988247c95
```

複製磁碟的UUID，打開`/etc/fstab`
``` console
$ vi etc/fstab
```

將硬碟加入`/etc/fstab`，`nofail`為在磁碟不可用時也能開機
``` console
UUID=ccff9e1e-b518-4ac5-aabd-e37988247c95 /data ext4 discard,defaults,nofail 0 2
```

接著重新啟動VM，並確認磁碟狀態，如果新磁碟有掛載上去就大功告成了～～

## 增加磁碟大小

### GCP端
在**Compute Engine** > **磁碟** 選擇要擴增的磁碟然後編輯它的大小，並儲存
> 大小只能增加不能減少

![Imgur](https://imgur.com/uajWt0O.png)

### Linux端
> 此方法只適用不是開機碟的磁碟

使用`resize2fs`調整磁碟掛載大小
``` console
resize2fs /dev/sdb
```

然後再使用`df -h`確認掛載大小就成功了～是不是超簡單！
