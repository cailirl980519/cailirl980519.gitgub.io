---
title: 那些封裝Flutter for Windows Desktop踩過的坑
date: 2021-10-05 17:58:52
categories: Flutter
tags: 
- App
- Flutter
- Dart
- Windows
- MSIX
---

當我用Flutter寫完一個App後，想要將這個App封裝成Window的MSIX應用程式，但發現網上Windows Desktop的教學屈指可數，且多數都是英文，於是就產生了想法把它記錄下來，把坑填好，以免以後要用到忘記踩過的坑！

### 前置作業

- 一個欲封裝的Flutter專案
- 一台安裝好Flutter環境的Windows電腦
- 一顆強壯的心

### 新增MSIX套件

本篇是運用Flutter Package中的[msix](https://pub.dev/packages/msix)套件來封裝Windows應用程式，所以首先～我們就先將msix套件新增至我們Flutter專案中的`pubspec.yaml`中。

```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  msix: ^2.2.3
```

### 建置Windwos Release版本應用

若Flutter專案中還沒有windows資料夾，則先輸入：

`flutter config --enable-windows-desktop`

接下來先來建置個windows的應用(.exe)，只需在終端機的Flutter專案的目錄下輸入：

`flutter build windows`

這時，建置好的Windows應用將會輸出至Flutter專案中**\build\windows\runner\Release**資料夾下。

> 若應用程式要上架至Windows Store，就不需要再額外進行數位簽署，上架Windows Store時會自動簽署。
> 若要將MSIX進行私有部署或測試，則須創建一個.pfx證書來將應用進行數位簽署。

*如不需數位簽署則直接跳至下面MSIX打包*

### 創建自簽名的.pfx證書

1. 安裝[OpenSSL](https://slproweb.com/products/Win32OpenSSL.html)

2. 將OpenSSL新增至環境變數
    e.g. `"C:\Program Files\OpenSSL-Win64\bin"`

3. 產生私鑰
    `openssl genrsa -out mykeyname.key 2048`

4. 使用私鑰產生自簽名證書(CSR)
    `openssl req -new -key mykeyname.key –subj “/CN=Comapny Name/O=Comapny Name/C=TW” -out mycsrname.csr`
    > + /CN: Company Name
    > + /O:  Organization Name
    > + /C:  Country Name

5. 使用私鑰及CSR文件生成自簽名證書(CRT)
    `openssl x509 -in mycsrname.csr -out mycrtname.crt -req -signkey mykeyname.key -days 10000`

6. 使用私鑰與CRT文件生成.pfx文件
    `openssl pkcs12 -export -out CERTIFICATE.pfx -inkey mykeyname.key -in mycrtname.crt`

### 將應用程式進行數位簽署

1. 將signtool新增至環境變數
    e.g. `C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0\x64`

2. 將.pfx證書簽署至exe
    `signtool sign /tr http://timestamp.digicert.com /td sha256 /fd sha256 /f CERTIFICATE.pfx /p 16595169 myapp.exe`
    > + /tr: 時間戳記
    > + /td: 時間戳記摘要演算法
    > + /fd: 檔案摘要演算法
    > + /f:  簽署憑證
    > + /p:  憑證密碼

3. 右鍵點擊.exe檔選取**內容**，確認**數位簽章**內有.pfx證書
<img src="/images/msix/image7.png" alt="signtool_verify" title="height='500'" height="500" />

### 將.pfx資訊新增至Flutter專案

- 確認.pfx資訊
    1. 於Windows PowerShell輸入
        `Get-PfxCertificate`
    2. 於FilePath[0]輸入.pfx路徑
    3. 跳出FilePath[1]留空Enter
    4. 輸入.pfx密碼
    5. 將輸出Subject記錄下來
![Imgur](https://i.imgur.com/G7bFMwg.png)

- 將`msix_config`新增至`pubspec.yaml`的最底部

```yaml
msix_config:
  display_name: myApp
  publisher_display_name: CompanyName
  identity_name: MyCompany.MySuite.MyApp
  msix_version: 1.0.0.0
  certificate_path: C:\<PathToCertificate>\<CERTIFICATE.pfx>
  certificate_password: 1234
  publisher: CN=Comapny Name, O=Comapny Name, C=TW
  logo_path: C:\<PathToIcon>\<Logo.png>
  start_menu_icon_path: C:\<PathToIcon>\<Icon.png>
  tile_icon_path: C:\<PathToIcon>\<Icon.png>
  vs_generated_images_folder_path: C:\<PathToFolder>\icons
  icons_background_color: transparent
  architecture: x64
  capabilities: 'internetClient,location,microphone,webcam'
```

將剛剛紀錄的Subject直接貼上`publisher`
> 大坑-1: 若排序錯了則會封裝失敗！！！


### MSIX打包

在終端機的Flutter專案的目錄下輸入：
`flutter pub run msix:create`

![Imgur](https://i.imgur.com/f4zNLCL.png)

Flutter專案中**\build\windows\runner\Release**資料夾下，會多產生一個msix檔。

#### 安裝未受信任應用程式

在msix封裝時，msix套件會自動簽署預設的測試簽章，但此時開啟.msix檔案還是會跳出此應用程式是未受信任的應用程式而無法安裝，如下圖：

![取自https://docs.microsoft.com/](https://docs.microsoft.com/zh-tw/windows/msix/images/msix-bad-cert.png)

此時，需要右鍵點擊.msix檔案再選取內容，將會看到**數位簽章**的標籤，點擊簽章再選取詳細資料。

![Imgur](https://i.imgur.com/Mjavwpf.png)

點擊**檢視憑證**
![Imgur](https://i.imgur.com/olEKJPH.png)

點擊**安裝憑證**
![Imgur](https://i.imgur.com/fX0wKil.png)

選擇**本機電腦**
![Imgur](https://i.imgur.com/3sqJikv.png)

選擇存放在**受信任的根憑證目錄**
![Imgur](https://i.imgur.com/A05bKqb.png)

點擊**完成**
![Imgur](https://i.imgur.com/SCVHYhx.png)

接著全部點確定，再開啟.msix檔案，就變為受信任的應用程式且可以安裝了～
![取自https://docs.microsoft.com/](https://docs.microsoft.com/zh-tw/windows/msix/images/msix-good-cert.png)