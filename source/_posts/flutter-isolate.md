---
title: 用Isolate實現Loading Widget不卡頓
date: 2021-12-16 15:14:06
categories: Flutter
tags:
- App
- Flutter
- Dart
- Isolate
---

> Flutter預設是單執行緒處理，在你還沒發覺UI變得卡頓時，你是不會發現的。

剛開始接觸Flutter時，完全沒有認真研究過他的執行序，認為只要把複雜或需要時間的工作丟進Future, async, await就解決一切了，直到...我發現UI變得超卡的！！於是就誕生了這篇文章...紀錄在頁面放上loading元件`CircularProgressIndicator()`並在背後做繁重的運算。

### Dart中的Isolate

如果不特別創建Isolate，我們的所有程式碼幾乎都是在預設的main Isolate中執行的，不管是使用Future, async, await，最後都還是在同一個Isolate上進行處理，如果我們給main Isolate太多繁重的工作，UI就會開始延遲，因此我們就需要一個新的Isolate來分擔繁重的工作。
Dart中的Isolate是不會共享記憶體的，所以必須透過Port來溝通，且Dart中的溝通是異步的，這就使得寫起來變得十分複雜啊...

### 預覽目標
![flutter-isolate](https://imgur.com/fPbyIxA.gif)

### 實作

首先，我們需要一個非常繁重的工作，於是我們就創一個超繁重的while迴圈，Isolate的工作內容都需寫在最上層（global）或是static funciton。

```dart
int countEven() {
  int num = Random().nextInt(100) + 1000000000;
  int count = 0;
  while (num > 0) {
    if (num % 2 == 0) {
      count++;
    }
    num--;
  }
  return count % 1000;
}
```
> 給他跑個十億多次，夠繁重了吧

接著，創造一個與Isolate溝通的通道`isolateCompute()`，分別會有兩個通訊端
- SendPort: 丟工作的Isolate的通訊端（傳送端）
- ReceivePort: 做工作的Isolate的通訊端（接收端）

當main Isolate帶著他的通訊端過來時：
1. 先創造一個做工作Isolate的通訊端
2. 並使用`sendPort.send(reveivePort.sendPort)`將兩個通訊端做連接
3. 連接上後接收端就可以用`receivePort`開始監聽訊息
4. 若有訊息包`MessagePackage`過來則開始處理
5. 將結果用臨時傳送器傳回main Isolate

``` dart
// 訊息包
class MessagePackage {
  SendPort sender; // 臨時傳送器
  dynamic msg; // 訊息
  MessagePackage(this.sender, this.msg);
}

isolateCompute(SendPort sendPort) {
  // 1. 創造一個做工作Isolate的通訊端
  ReceivePort receivePort = ReceivePort();
  // 2. 將兩個通訊端做連接
  sendPort.send(receivePort.sendPort);
  // 3. 連接上後接收端就可以開始監聽訊息
  receivePort.listen((package) {
    // 4. 若有訊息過來則開始處理
    MessagePackage _msg = package as MessagePackage;
    print(_msg.msg); // output: a message to isolate
    int r = countEven();
    // 5. 將結果用臨時傳送器傳回去
    _msg.sender.send(r);
  });
}
```

創建完Isolate後，我們就可以在頁面上加上`_executeIsolate()`來呼叫Isolate做事
1. 創建一個通訊端
2. 將通訊端傳至Isolate，並等待連接上，即變成傳送端
3. 建立好通道後即可使用傳送端將訊息包（臨時傳送器＋訊息）傳至Isolate
4. 回傳結果
5. 關閉臨時傳送器
6. 關閉通訊端
```dart
int? _counter;
bool _computing = false;

Isolate? isolate;
SendPort? isolateSender;
void _executeIsolate() async {
  setState(() {
    _computing = true;
  });
  // 1. 創建一個通訊端
  ReceivePort receivePort = ReceivePort();
  // 2. 將通訊端傳至Isolate，並等待連接上
  isolate =
      await Isolate.spawn<SendPort>(isolateCompute, receivePort.sendPort);
  isolateSender = await receivePort.first;
  // 3. 用傳送端將訊息包傳至Isolate
  ReceivePort _temp = ReceivePort();
  isolateSender!.send(MessagePackage(_temp.sendPort, "a message to isolate"));
  // 4. 回傳結果
  int r = await _temp.first;
  setState(() {
    _counter = r;
  });
  // 5. 關閉臨時傳送器
  _temp.close();
  // 6. 關閉通訊端
  receivePort.close();
  setState(() {
    _computing = false;
  });
}
```

最後做個按鈕來觸發`_executeIsolate()`，並在運算時把`CircularProgressIndicator()`加入頁面

``` dart
@override
Widget build(BuildContext context) {
  int i = _counter ?? 0;
  return Scaffold(
    appBar: AppBar(
      title: const Text("Flutter Demo Home Page"),
    ),
    body: Center(
      child: Container(
        alignment: Alignment.center,
        width: MediaQuery.of(context).size.width * 0.5,
        height: MediaQuery.of(context).size.width * 0.5,
        color: Colors.grey[100],
        child: _computing
            ? const CircularProgressIndicator()
            : Text(
                "$i",
                style: Theme.of(context).textTheme.headline1,
              ),
      ),
    ),
    floatingActionButton: FloatingActionButton(
      onPressed: _executeIsolate,
      tooltip: 'Change',
      child: _computing
          ? const CircularProgressIndicator()
          : const Icon(Icons.swap_horiz_rounded),
    ),
  );
}
```