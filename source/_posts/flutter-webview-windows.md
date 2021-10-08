---
title: Flutter的Windows套件 - webview_windows
date: 2021-10-07 14:17:28
categories: Flutter
tags:
- Flutter
- Dart
- Windows
- WebView
---

### 前言

WebView在Flutter的iOS及Android端支援已非常完善，但在Windows及Mac端卻始終沒有支援，在2019年，就有網友在Github提出何時會支援電腦桌面端的WebView套件，但始終沒有消息...

終於在兩年後，隨著微軟推出了WebView2，終於也有大神在Flutter為Windows開發了[Webview](https://pub.dev/packages/webview_windows)套件！！！

今天就來好好了解該怎麼使用這個套件吧～

### 配置需求

+ 用戶端的電腦
    - [WebView2 執行階段](https://developer.microsoft.com/zh-tw/microsoft-edge/webview2/)
    > 當然要debug的話也是要安裝
    - Windows 10 1809+

+ 開發端的電腦
    - [Visual Studio 2019](https://visualstudio.microsoft.com/zh-hant/downloads/)
    - Windows 10 SDK 2004+ (10.0.19041.0)

#### Tips
查看看Windows的版本：**開始** > **設定** > **系統** > **關於**
![windows_ver](https://i.imgur.com/yZKv4QW.png)

### 安裝

首先，當然就是將套件放進`pubspec.yaml`中

``` yaml
dev_dependencies:
  webview_windows: ^0.0.8
```

然後安裝套件
``` console
$ flutter pub get
```
再將它匯入專案中
``` dart
import 'package:webview_windows/webview_windows.dart' as win;
```

### 使用

![source: https://lottiefiles.com/jpdvuc5tor](https://i.imgur.com/yOKNVnq.gif)
突然沒了動力...過兩天再寫~