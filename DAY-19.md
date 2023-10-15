# 【鐵人賽】DAY-19 Flutter 顯示即時資料

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-09-進入-Golang-『一』-02.jpg)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1098/)

## 前言

回到台灣了
終於不用思思念念怕鐵人賽斷更🤣
昨天在 `Golang` 那邊建立了一個 `WebSocket` 通道
雖然還沒把資料放進來
但可以先小跳一點到 `Flutter` 這邊
先把 `WebSocket` 打通
最後再把即時價格透過這個通道送下來
這樣就可以第一手顯示最新的價格～

## 安裝

這邊我們使用的 `dart` 套件是 [web\_socket\_channel](https://pub.dev/packages/web_socket_channel)
沒有非用它不可的原因
僅是我用起來比較順手
硬要比較 Like 數量的話，還有替代方案是 [socket\_io\_client](https://pub.dev/packages/socket_io_client)
這篇文章就先使用 `web_socket_channel: ^2.4.0`
首先在 `pubspec.yaml` 的 `dependencies` 加入以下

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.1.0
  web_socket_channel: ^2.4.0
```

```bash
# 記得要安裝
flutter pub get --no-example
```

## 包裝

作者提供的範例實在是太少（下圖）
按照他這樣寫不是不行
但絕對不會是一個合格的實作
幫各位踩了一些坑🤣

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-19-Flutter-顯示即時資料-01.png)

### 建立連線

這邊把連線包成一個方法
可以看得出來，比範例還要多一些東西
後面會解釋為什麼需要
但是裡面還沒有實作資料進來的部分哦

```dart
import 'package:web_socket_channel/io.dart';

void initialWS() {
  _channel = IOWebSocketChannel.connect(Uri.parse('ws://127.0.0.1:8080/ws/time'), pingInterval: const Duration(seconds: 1));
  _channel!.stream.listen(
    (message) {
      if (!mounted) {
        return;
      }
    },
    onDone: () {
      if (mounted) {
        Future.delayed(const Duration(milliseconds: 1000)).then((value) {
          _channel!.sink.close();
          initialWS();
        });
      }
    },
    onError: (error) {},
  );
}
```

### initState

還記得我們在 [DAY-11](https://tocandraw.com/2023-ironman/934/) 有提到 `Flutter` 是有兩種 `Widget` 的
分別是 `StatelessWidget`、`StatefulWidget`
很明顯這個串流即時價格的功能會是有狀態的
生命週期應該會是進入這個頁面，然後建立一個通道
在 `StatefulWidget` 中，初始化的方法就是 `initState()`

```dart
@override
void initState() {
  initialWS();
  super.initState();
}
```

跟大部分 `OOP` 語言一樣
透過 `override`，我們可以在超類的 `initState` 裡面去做一些變更
這邊我們所做的就是把連線這件事加入

### dispose

`dispose()` 也是 `StatefulWidget` 獨有的方法
他的作用是當這個畫面不在被渲染時就會執行
直白的說就是使用者切換到其他畫面時
同樣我們要來 `override`

```dart
@override
void dispose() {
  _channel!.sink.close();
  super.dispose();
}
```

很容易看得出來，這邊就是很簡單做了主動關閉
不關也行，但後端是我們自己寫的
能關就關嘛🤓

## 整合 Golang WebSocket

我們在 [DAY-18](https://tocandraw.com/2023-ironman/1083/) 有在 `Golang` 這邊開了一個暫時性的 `WebSocket`
下面這個 `URL`

```markup
/ws/time

# 給他 time 就會回覆 Current time
```

我們先暫時把 `APP`
改成以下這樣

```dart
Future<String> currentTime = Future<String>.value('');

@override
Widget build(BuildContext context) {
  return MaterialApp(
    home: Scaffold(
      appBar: AppBar(
        title: const Text('Try WebSocket'),
      ),
      body: FutureBuilder<String>(
        future: currentTime,
        builder: (context, snapshot) {
          if (snapshot.hasData && snapshot.data!.isNotEmpty) {
            return Center(
              child: Text(snapshot.data!),
            );
          }
          return const Center(
            child: CircularProgressIndicator(
              color: Colors.black,
            ),
          );
        },
      ),
    ),
  );
}
```

畫面上應該如下

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-19-Flutter-顯示即時資料-02.png)

然後我們再加一個方法
可以送出 `time` 給 `Golang` 那邊

```dart
void sendReq() {
  _channel!.sink.add('time');
}
```

好像漏了什麼
需要交互，所以還是把按鈕加回來

```dart
appBar: AppBar(
  title: const Text('Try WebSocket'),
  actions: [
    IconButton(
      icon: const Icon(Icons.timer),
      onPressed: sendReq,
    ),
  ],
),
```

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-19-Flutter-顯示即時資料-03.png)

合起來看看
順便砍掉一些多餘字元

```dart
@override
Widget build(BuildContext context) {
  return MaterialApp(
    home: Scaffold(
      appBar: AppBar(
        title: const Text('Try WebSocket'),
        actions: [
          IconButton(
            icon: const Icon(Icons.timer),
            onPressed: sendReq,
          ),
        ],
      ),
      body: FutureBuilder<String>(
        future: currentTime,
        builder: (context, snapshot) {
          if (snapshot.hasData && snapshot.data!.isNotEmpty) {
            return Center(
              child: Text(
                snapshot.data!.toString().substring(0, 33),
                style: const TextStyle(fontSize: 20),
              ),
            );
          }
          return const Center(
            child: CircularProgressIndicator(
              color: Colors.black,
            ),
          );
        },
      ),
    ),
  );
}
```

### 按按看

試著按按看右上角的碼表
畫面中央應該出現 `Golang` 透過 `WebSocket` 送過來的時間

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-19-Flutter-顯示即時資料-04.png)

看看 `terminal`
這不是呼叫 `API`
而是實打實的 `WebSocket` 的訊息交換

```bash
flutter: send time req
flutter: receive time: Current time: 2023-10-04 21:22:48.887227 +0800 CST m=+1444.888261751
flutter: send time req
flutter: receive time: Current time: 2023-10-04 21:22:49.50685 +0800 CST m=+1445.507884210
flutter: send time req
flutter: receive time: Current time: 2023-10-04 21:22:50.115684 +0800 CST m=+1446.116718293
flutter: send time req
flutter: receive time: Current time: 2023-10-04 21:22:50.659663 +0800 CST m=+1446.660698085
```

## 總結

雖然今天這樣的展示
有那麼一點不像是 `WebSocket`
但我們知道其實這不是呼叫 `API`
明天就可以把正式的資料丟進來讓他一直一直跳動了～
