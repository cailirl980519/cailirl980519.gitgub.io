---
title: Flutter的Windows套件 - webview_windows (2)
date: 2021-10-13 14:37:03
categories: Flutter
tags:
- App
- Flutter
- Dart
- Windows
- Edge Webview2
---

前幾天介紹了webview_windows的[基本使用](https://cailirl980519.github.io/2021/10/07/flutter-webview-windows/)，我們已經學會了如何在Flutter App裡開啟網頁，但通常如果是開啟自己的網頁的話，可能有些Web功能就需要跟Flutter端溝通。
今天要來學習如何運用[webview_windows](https://pub.dev/packages/webview_windows)來接收網頁的訊息及從Flutter傳訊息至Web端。

### 前置作業

- 引入`webview_windows`套件
- 創建一個需要溝通的html檔
> 以下是這次範例所使用的html
``` html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC" crossorigin="anonymous">
	<title>WebView Sample for Windows</title>
</head>
<body>
    <h1 class="text-center">Communicate With Flutter</h1>
    <div>
        <script src="https://unpkg.com/@lottiefiles/lottie-player@latest/dist/lottie-player.js"></script>
        <lottie-player class="mx-auto" src="https://assets1.lottiefiles.com/packages/lf20_zwykwl1i.json"  background="transparent"  speed="1"  style="width: 300px; height: 300px;"  loop autoplay></lottie-player>
    </div>
    <div class="row justify-content-around">
        <div class="col-5">
            <h3>Example 1.</h3>
            <input id="web_message" class="w-100 mb-2" type="text" value="Hi! I'm web message.">
            <button type="button" class="btn btn-primary" onclick="sendMessage()">Send Message to Flutter</button>
        </div>
        <div class="col-5">
            <h3>Example 2.</h3>
            <p>Message from Dart:</p>
            <textarea id="flutter" class="w-100" disabled="true" style="height: 30vw;"></textarea>
        </div>
    </div>
</body>
</html>
```
### Web to Flutter - 將Web訊息傳至Flutter

#### Web端
在JavaScipt內加入function `sendMessage()`

``` js
const message = document.getElementById("web_message") // 輸入文字的input元件
function sendMessage() {
    // 若無Flutter通道則不動作
    if (window.chrome === undefined && window.chrome.webview === undefined)
        return
    // 將訊息以json格式傳送
    window.chrome.webview.postMessage({"message": message.value})
}
```

#### Flutter端
先宣告`WebviewController`
``` dart
import 'package:webview_windows/webview_windows.dart' as wweb;

final _winController = wweb.WebviewController();
```

並寫一個`initWindowsWebView()`的function來初始Webview，並在Webview載入完成時監聽Web端訊息
``` dart
Future<void> initWindowsWebView() async {
    if (!Platform.isWindows) return;
    await _winController.initialize();
    String data = await rootBundle.loadString("assets/webpages/index.html");
    await _winController.loadStringContent(data);
    while (!_winController.value.isInitialized) {
        sleep(Duration(milliseconds: 200));
    }
    setState(() {});
    // 等待Webview載入完成
    await for (wweb.LoadingState value in _winController.loadingState) {
        if (value.toString() == "LoadingState.navigationCompleted") {
            // 監聽Web端訊息
            _winController.webMessage.listen((args) async {
                // args為接收到的訊息(e.g. {"message": "Hi! I'm web message."})
                // 收到訊息動作（彈出對話框）
                showDialog(
                    context: context,
                    builder: (context) {
                        return AlertDialog(
                            title: Text("Message from Web"),
                            content: Text("${args["message"]}"),
                        );
                    },
                );
            });
        }
    }
}
```

最後在`initState()`時呼叫`initWindowsWebView()`，並在離開時關閉`WebviewController`
```dart
@override
void initState() {
    super.initState();
    initWindowsWebView();
}

@override
void dispose() {
    super.dispose();
    _winController.dispose(); // 增加這行來關閉WebviewController
}
```

### Flutter to Web - 將Flutter訊息傳至Web

#### Web端
在JavaScipt加入監聽Flutter訊息的Code
``` js
const textarea = document.getElementById("flutter") // 顯示Flutter訊息的textarea元件
// 若無Flutter通道則不監聽
if (window.chrome !== undefined && window.chrome.webview !== undefined) {
    window.chrome.webview.addEventListener('message', function(e) {
        // 若收到到訊息動作
        textarea.innerHTML += (e.data.message + "\n")
    })
}
```
#### Flutter端

宣告一個`TextEditingController`來輸入訊息，並在離開時關閉
``` dart
final _textController = TextEditingController(text: "Hi! I'm flutter message.");

@override
void dispose() {
    super.dispose();
    _textController.dispose(); // 增加這行來關閉TextEditingController
}
```

新增一個`IconButton`元件來傳送訊息至Web
``` dart
IconButton(
    icon: Icon(Icons.send),
    onPressed: () {
        showDialog(
            context: context,
            builder: (context) {
                return AlertDialog(
                    title: Text("Send Message to Web"),
                    content: TextField(
                        controller: _textController,
                    ),
                    actions: [
                        TextButton(
                            child: Text("Send"),
                            onPressed: () {
                                // 將欲傳送至Web的訊息宣告為data
                                Map data = {"message": _textController.text};
                                // 將data傳送至Web
                                _winController.postWebMessage('${jsonEncode(data)}');
                                Navigator.of(context).pop();
                            },
                        ),
                    ],
                );
            },
        );
    },
)
```

### 大坑1 - UTF8編碼問題
在webview_windows 0.0.8（目前最新）這個版本中，尚未對UTF8編碼做特殊處理，也就是如果在溝通上使用中文字的話則會出現問題，所以我就研究了一下套件源碼，經歷了千辛萬苦(?，終於讓套件用中文溝通了～

首先，我們先進入webview_windows的檔案位置，通常會在Flutter SDK位置下的資料夾內
*e.g. .../flutter/.pub-cache/hosted/pub.dartlang.org/webview_windows-0.0.8/windows/*

接著，找到`webview_bridge.cc`檔案並修改裡面的`towstring` function
``` c++
std::wstring towstring(std::string_view str) {
  if (str.empty()) {
    return std::wstring();
  }
  int utf16Length = ::MultiByteToWideChar(CP_UTF8, MB_ERR_INVALID_CHARS, str.data(), static_cast<int>(str.length()), nullptr, 0);
  if (utf16Length == 0) {
    return std::wstring();
  }
  std::wstring utf16;
  utf16.resize(utf16Length);
  int convertLength = ::MultiByteToWideChar(CP_UTF8, MB_ERR_INVALID_CHARS, str.data(), static_cast<int>(str.length()), utf16.data(), utf16Length);
  if (convertLength == 0) {
    return std::wstring();
  }
  return utf16;
}
```
> 修改後，此function會將傳至Web的值變為正確的UTF8編碼

接下來，我們再來修改`webview.cc`中的`webview_->add_WebMessageReceived`
``` c++
webview_->add_WebMessageReceived(
  Callback<ICoreWebView2WebMessageReceivedEventHandler>(
      [this](ICoreWebView2* sender,
              ICoreWebView2WebMessageReceivedEventArgs* args) -> HRESULT {
        wil::unique_cotaskmem_string wmessage;
        if (web_message_received_callback_ &&
            args->get_WebMessageAsJson(&wmessage) == S_OK) {
          const std::string message =  CW2A(wmessage.get(), CP_UTF8); // 加入CP_UTF8指定UTF8編碼
          web_message_received_callback_(message);
        }

        return S_OK;
      })
      .Get(),
  &event_registrations_.web_message_received_token_);
```
> 修改後，Web接收到的值經過C++後中文就不會變亂碼了

### Bonus - 回傳Web的console.log訊息至Flutter端
再測試時，常常需要用到`console.log`來確認程式碼有沒有問題，所以我們就可以直接劫持console.log訊息，將他傳至Flutter端。
#### Web端
``` js
console.log = function (txt) {
    let messages = []
    // arguments為console.log輸出訊息
    for (var i = 0; i < arguments.length; i++) {
        messages.push(arguments[i]) // 將訊息存至messages
    }
    // 將messages傳至Flutter端
    window.chrome.webview.postMessage(
        { 
            "method": "console", // 宣告來源，方便Flutter端判斷
            "message": messages.join(" ") 
        }
    )
}
```
> `console.log`被劫持後就不會印在主控台(Console)上了哦！畢竟它被綁架了麻～

#### Flutter端
我們一樣在`initWindowsWebView()`內監聽，只是在多加判斷它是不是console的訊息，如果是的話則輸出至Flutter Console。
``` dart
await for (wweb.LoadingState value in _winController.loadingState) {
    if (value.toString() == "LoadingState.navigationCompleted") {
        // 監聽Web端訊息
        _winController.webMessage.listen((args) async {
            // args為接收到的訊息(e.g. {"message": "Hi! I'm web message."})
            if (args["method"] == "console") {
                // 若為console訊息動作
                print(args["message"]);
            } else {
                // 收到訊息動作（彈出對話框）
                showDialog(
                    context: context,
                    builder: (context) {
                        return AlertDialog(
                            title: Text("Message from Web"),
                            content: Text("${args["message"]}"),
                        );
                    },
                );
            }
            
        });
    }
}
```

### 感動到哭的成果時間

<script src="https://unpkg.com/@lottiefiles/lottie-player@latest/dist/lottie-player.js"></script>
<lottie-player src="https://assets9.lottiefiles.com/packages/lf20_xkbhgbld.json"  background="transparent"  speed="1"  style="width: 300px; height: 300px;"  loop autoplay></lottie-player>

*忘記備份成果了，等有Windows電腦後再回來放...*

### Full Code

> 完整程式碼請至[sample_windows_webview](https://github.com/cailirl980519/sample_windows_webview)

#### index.html
``` html
<!DOCTYPE html>
<html>
<head>
	<meta charset="utf-8">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC" crossorigin="anonymous">
	<title>WebView Sample for Windows</title>
</head>
<body>
    <h1 class="text-center">Communicate With Flutter</h1>
    <div>
        <script src="https://unpkg.com/@lottiefiles/lottie-player@latest/dist/lottie-player.js"></script>
        <lottie-player class="mx-auto" src="https://assets1.lottiefiles.com/packages/lf20_zwykwl1i.json"  background="transparent"  speed="1"  style="width: 300px; height: 300px;"  loop autoplay></lottie-player>
    </div>
    <div class="row justify-content-around">
        <div class="col-5">
            <h3>Example 1.</h3>
            <input id="web_message" class="w-100 mb-2" type="text" value="Hi! I'm web message.">
            <button type="button" class="btn btn-primary" onclick="sendMessage()">Send Message to Flutter</button>
        </div>
        <div class="col-5">
            <h3>Example 2.</h3>
            <p>Message from Dart:</p>
            <textarea id="flutter" class="w-100" disabled="true" style="height: 30vw;"></textarea>
        </div>
    </div>
</body>
<script>
    const message = document.getElementById("web_message")
    function sendMessage() {
        if (window.chrome === undefined && window.chrome.webview === undefined)
            return
        window.chrome.webview.postMessage({"message": message.value})
    }
    const textarea = document.getElementById("flutter")
    if (window.chrome !== undefined && window.chrome.webview !== undefined) {
        window.chrome.webview.addEventListener('message', function(e) {
            textarea.innerHTML += (e.data.message + "\n")
        })
    }
</script>
</html>
```

#### index.html
``` dart
import 'dart:convert';
import 'dart:io';

import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:lottie/lottie.dart';
import 'package:webview_windows/webview_windows.dart' as wweb;

class WebView2Page extends StatefulWidget {
  const WebView2Page({Key? key}) : super(key: key);

  @override
  _WebView2PageState createState() => _WebView2PageState();
}

class _WebView2PageState extends State<WebView2Page> {
  final _winController = wweb.WebviewController();
  final _textController =
      TextEditingController(text: "Hi! I'm flutter message.");

  Future<void> initWindowsWebView() async {
    if (!Platform.isWindows) return;
    await _winController.initialize();
    String data = await rootBundle.loadString("assets/webpages/index.html");
    await _winController.loadStringContent(data);
    while (!_winController.value.isInitialized) {
      sleep(Duration(milliseconds: 200));
    }
    setState(() {});
    await for (wweb.LoadingState value in _winController.loadingState) {
      if (value.toString() == "LoadingState.navigationCompleted") {
        _winController.webMessage.listen((args) async {
          showDialog(
            context: context,
            builder: (context) {
              return AlertDialog(
                title: Text("Message from Web"),
                content: Text("${args["message"]}"),
              );
            },
          );
        });
      }
    }
  }

  Widget webView() {
    if (!Platform.isWindows)
      return Center(
        child: Text("not Support"),
      );
    if (!_winController.value.isInitialized)
      return Center(
        child: Lottie.network(
            "https://assets9.lottiefiles.com/datafiles/bEYvzB8QfV3EM9a/data.json"),
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
    _textController.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("Sample2"),
        actions: [
          IconButton(
            icon: Icon(Icons.send),
            onPressed: () {
              showDialog(
                context: context,
                builder: (context) {
                  return AlertDialog(
                    title: Text("Send Message to Web"),
                    content: TextField(
                      controller: _textController,
                    ),
                    actions: [
                      TextButton(
                        child: Text("Send"),
                        onPressed: () {
                          Map data = {
                            "message": _textController.text,
                          };
                          print(data);
                          _winController.postWebMessage('${jsonEncode(data)}');
                          Navigator.of(context).pop();
                        },
                      ),
                    ],
                  );
                },
              );
            },
          )
        ],
      ),
      body: webView(),
    );
  }
}
```
