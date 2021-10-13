---
title: Flutter的Windows套件 - webview_windows (1)
description: Flutter webview_windows的基本使用
date: 2021-10-07 14:17:28
categories: Flutter
tags:
- Flutter
- Dart
- Windows
- WebView
- Edge Webview2
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
import 'package:webview_windows/webview_windows.dart' as wweb;
```

### 使用

首先，我們先宣告一個Controller來控制WebView
``` dart
final _winController = wweb.WebviewController();
```
並寫一個初始WebView的function
``` dart
Future<void> initWindowsWebView() async {
  if (!Platform.isWindows) return;
  await _winController.initialize();  // 初始WebView
  await _winController.loadUrl(
      "https://developer.microsoft.com/en-us/microsoft-edge/webview2/"); // 載入頁面
  while (!_winController.value.isInitialized) { // 等待初始完成
    sleep(Duration(milliseconds: 200));
  }
  setState(() {}); // 重整頁面UI
}
```

再將`initWindowsWebView()`放至`initState()`中
``` dart
@override
void initState() {
  super.initState();
  initWindowsWebView();
}
```

然後切記!要在離開頁面時關閉WebviewController
``` dart
@override
void dispose() {
  super.dispose();
  _winController.dispose();
}
```

寫完動作後，我們再來寫一個Widget來顯示WebView
``` dart
Widget webView() {
  if (!Platform.isWindows)
    return Center(
      child: Text("not Support"),
    );
  if (!_winController.value.isInitialized)
    return Center(
      child: CircularProgressIndicator(), // 未初始完成顯示loading圖示
    );
  return Container(
    child: wweb.Webview(_winController),
  );
}
```

最後，再將Wiget放進頁面就完成了~
``` dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: Text("WebView"),
    ),
    body: webView(),
  );
}
```

### 另人激動的成果時間

![webview preview](https://i.imgur.com/ZfztlMU.gif)

### Full Code

> 完整程式碼請至[sample_windows_webview](https://github.com/cailirl980519/sample_windows_webview)

``` dart
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:lottie/lottie.dart';
import 'package:webview_windows/webview_windows.dart' as wweb;

class WebViewPage extends StatefulWidget {
  const WebViewPage({Key? key}) : super(key: key);

  @override
  _WebViewPageState createState() => _WebViewPageState();
}

class _WebViewPageState extends State<WebViewPage> {
  final _winController = wweb.WebviewController();

  Future<void> initWindowsWebView() async {
    if (!Platform.isWindows) return;
    await _winController.initialize();  // 初始WebView
    await _winController.loadUrl(
        "https://developer.microsoft.com/en-us/microsoft-edge/webview2/"); // 載入頁面
    while (!_winController.value.isInitialized) { // 等待初始完成
      sleep(Duration(milliseconds: 200));
    }
    setState(() {}); // 重整頁面UI
  }

  Widget webView() {
    if (!Platform.isWindows)
      return Center(
        child: Text("not Support"),
      );
    if (!_winController.value.isInitialized)
      return Center(
        child: CircularProgressIndicator(), // 未初始完成顯示loading圖示
      );
    return Container(
      child: wweb.Webview(_winController),
    );
  }

  @override
  void initState() {
    super.initState();
    initWindowsWebView();
  }

  @override
  void dispose() {
    super.dispose();
    _winController.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Sample1"),
      ),
      body: webView(),
    );
  }
}
```
