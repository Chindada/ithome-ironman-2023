# 【鐵人賽】DAY-20 跳動吧！即時價格

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【Flutter】Xcode-DT_TOOLCHAIN_DIR-cannot-be-used-to-evaluate-LIBRARY_SEARCH_PATHS-01.jpg)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1121/)

## 前言

昨天先把 `Flutter` 跟 `Golang` 的 `WebSocket` 打通了
但僅是做了一個一來一回的交互
還沒有把真實的價格給串上來
今天會把 `Golang` 收到的價格，做一點處理
再往 `WebSocket` 裡頭送
最後做一點美化～

## Golang

今天主要會把重點放在 `Flutter` 這邊
所以這邊的寫法會稍微簡陋一些
正常如果要部署到正式環境
需要再寫個更好一些～
昨天在 `WebSocket Server` 這邊
我們主要是監聽 `Client` 是否有送出 `time` 這個訊息
但既然是要串流
我們就不管三七二十一
`Client` 一連上來，馬上就直接把資料灌進去

```go
handler := gin.New()
handler.Use(gin.Recovery())
wsRoute := handler.Group("/ws")
wsRoute.GET("/time", func(c *gin.Context) {
 w := httpserver.NewWSRouter(c, logger)
 forwardChan := make(chan []byte)
 go func() {
  for {
                    # 在這邊，把資料源源不絕送出去
                    # 無論使用者有沒有送任何資料
  }
 }()
 w.ReadFromClient(forwardChan)
})
```

我們先開一個放即時價格的 `Channel`

```go
priceChan := make(chan []byte)
```

然後再 `WebSocket` 的 `for loop`
收到就用 `SendToClient` 把資料送出去

```go
for {
 select {
 case <-w.Ctx().Done():
  return
 default:
  w.SendToClient(<-priceChan)
 }
}
```

那資料從哪裡來呢
還記得前天有從 `RabbitMQ` 的 `routing mode` 取得的資料嗎？
因為在 `WebSocket` 起來的時候就已經開始從 `priceChan` 這個 `Channel` 取資料
所以不用怕會阻塞

```go
for {
 d, opened := <-delivery
 if !opened {
  return
 }
 priceChan <- d.Body
}
```

至此就已經完成送資料

## Flutter

細心的同學應該已經發現
剛剛 `Golang` 送出來的時候
是用 `Protobuf` 的資料型態，也就是 `[]byte`
所以我們在 `Flutter` 如果直接收
將會是一段看不懂的 `bytes`
這邊也會需要做一個 `Unmarshal`
但首先，我們來編譯一下 `Flutter Protobuf`

```bash
protoc \
    --dart_out=./lib/pb \
    --proto_path=./trade-protobuf/protos/v3/app \
    --proto_path=./trade-protobuf/protos/v3/forwarder \
    ./toc-trade-protobuf/protos/v3/*/*.proto
```

如果直接執行
會報錯哦

```bash
protoc-gen-dart: program not found or is not executable
Please specify a program using absolute path or make sure the program is available in your PATH system variable
--dart_out: protoc-gen-dart: Plugin failed with status code 1.
```

因為就像 `Python` 一樣
還需要特別裝一下 `Dart` 在 `Protobuf` 上的 `Plugin`

```bash
$dart pub global activate protoc_plugin
+ collection 1.18.0ies...
+ fixnum 1.1.0
+ meta 1.10.0
+ path 1.8.3
+ protobuf 3.1.0
+ protoc_plugin 21.1.1
Building package executables...
Built protoc_plugin:protoc_plugin_bazel.
Built protoc_plugin:protoc_plugin.
Installed executable protoc-gen-dart.
Activated protoc_plugin 21.1.1.
```

在編譯一次
應該可以在 lib 中看到一群 `*pb.dart`
但有沒有發現怪怪的
一堆紅字

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-20-跳動吧！即時價格-01.png)

那是因為 `Flutter` 也需要 `Protobuf` 的 `Dependency`
來加一下

```yaml
dependencies:
  flutter:
    sdk: flutter
  http: ^1.1.0
  web_socket_channel: ^2.4.0
  protobuf: ^3.1.0
  grpc: ^3.2.4
```

順便連 `grpc` 都加進去了
其實是用不到
因為在 `Flutter` 這邊用不到 `rpc`
只會用到 `Protobuf` 的格式

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-20-跳動吧！即時價格-02.png)

這邊就要新增一個 `class` 來處理 `protobuf` 的資料

```dart
class RealTimeFutureTick {
  RealTimeFutureTick(
    this.code,
    this.open,
    this.underlyingPrice,
    this.bidSideTotalVol,
    this.askSideTotalVol,
    this.avgPrice,
    this.close,
    this.high,
    this.low,
    this.amount,
    this.totalAmount,
    this.volume,
    this.totalVolume,
    this.tickType,
    this.chgType,
    this.priceChg,
    this.pctChg,
    this.simtrade,
  );

  RealTimeFutureTick.fromProto(pb.FutureRealTimeTickMessage tick) {
    code = tick.code;
    open = tick.open;
    underlyingPrice = tick.underlyingPrice;
    bidSideTotalVol = tick.bidSideTotalVol.toInt();
    askSideTotalVol = tick.askSideTotalVol.toInt();
    avgPrice = tick.avgPrice;
    close = tick.close;
    high = tick.high;
    low = tick.low;
    amount = tick.amount;
    totalAmount = tick.totalAmount;
    volume = tick.volume.toInt();
    totalVolume = tick.totalVolume.toInt();
    tickType = tick.tickType.toInt();
    chgType = tick.chgType.toInt();
    priceChg = tick.priceChg;
    pctChg = tick.pctChg;
    if (tick.priceChg > 0) {
      changeType = '↗️';
    } else if (tick.priceChg < 0) {
      changeType = '↘️';
    } else {
      changeType = '';
    }
  }

  String? code;
  DateTime? tickTime;
  num? open;
  num? underlyingPrice;
  num? bidSideTotalVol;
  num? askSideTotalVol;
  num? avgPrice;
  num? close;
  num? high;
  num? low;
  num? amount;
  num? totalAmount;
  num? volume;
  num? totalVolume;
  num? tickType;
  num? chgType;
  num? priceChg;
  num? pctChg;
  num? simtrade;
  bool? combo = false;
  String? changeType;
}
```

## 總結

不小心沒有切好斷點
今天先定義好格式
明天會再來繼續把收到的資料顯示出來
