# 【鐵人賽】DAY-27 完成下單介面『二』

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-26-完成下單介面『一』-02.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1308/)

## 前言

剩下最後四篇
就剩下把想要的介面刻到畫面上
就能夠完成專屬自己的下單介面
今天會從 `Flutter` 呼叫 `Golang` 上的交易 `API`
然後確認券商系統中是否有存在這筆下單
完成這一次主題最重要的一段資料流

## RESTful API

首先，來盤一下目前有哪幾隻 `API`

```sh
[GIN-debug] PUT    /trade/future/buy         --> main.main.func2 (2 handlers)
[GIN-debug] PUT    /trade/future/sell        --> main.main.func3 (2 handlers)
[GIN-debug] PUT    /trade/future/cancel      --> main.main.func4 (2 handlers)
```

### 買

- URL: **/trade/future/buy**

- Method: **PUT**

- Body: 如下所示

```json
{
    "code": "MXFJ3",
    "base_order": {
        "price": 16660,
        "quantity": 1
    }
}
```

### 賣

- URL: **/trade/future/sell**

- Method: **PUT**

- Body: 如下所示

```json
{
    "code": "MXFJ3",
    "base_order": {
        "price": 16660,
        "quantity": 1
    }
}
```

### 取消

- URL: **/trade/future/sell**

- Method: **PUT**

- Body: 如下所示

```json
{
    "order_id": "d1c55828",
}
```

## Flutter

目前的畫面應該是下面這樣
只有價格在跳動
但是沒有按鈕可以操作

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-27-完成下單介面『二』-01.png)

現在就可以一步一步來
首先我們先把呼叫 `API` 的方法寫好

### Class

還記得前幾天利用 `Postman` 測試 `API` 的結果嗎
就像下面這張圖

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-27-完成下單介面『二』-02.png)

這邊我們會利用這個[工具](https://javiercbk.github.io/json_to_dart/)把這隻 `API` 的 `Input`、`Output`
很方便地從 `JSON` -> `Dart`

#### OrderRequest

從 `Input` 開始，我們設計好的 `Body` 如下

```json
{
    "code": "MXFJ3",
    "base_order": {
        "price": 16660,
        "quantity": 1
    }
}
```

透過工具很輕鬆就變成以下

```dart
class OrderRequest {
  String? code;
  BaseOrder? baseOrder;

  OrderRequest({this.code, this.baseOrder});

  OrderRequest.fromJson(Map<String, dynamic> json) {
    code = json['code'];
    baseOrder = json['base_order'] != null
        ? new BaseOrder.fromJson(json['base_order'])
        : null;
  }

  Map<String, dynamic> toJson() {
    final Map<String, dynamic> data = new Map<String, dynamic>();
    data['code'] = this.code;
    if (this.baseOrder != null) {
      data['base_order'] = this.baseOrder!.toJson();
    }
    return data;
  }
}

class BaseOrder {
  int? price;
  int? quantity;

  BaseOrder({this.price, this.quantity});

  BaseOrder.fromJson(Map<String, dynamic> json) {
    price = json['price'];
    quantity = json['quantity'];
  }

  Map<String, dynamic> toJson() {
    final Map<String, dynamic> data = new Map<String, dynamic>();
    data['price'] = this.price;
    data['quantity'] = this.quantity;
    return data;
  }
}
```

同樣的，`Response` 也是一樣

```json
{
    "order_id": "d1c55828",
    "status": 1
}
```

```dart
class TradeResponse {
  String? orderId;
  int? status;

  TradeResponse({this.orderId, this.status});

  TradeResponse.fromJson(Map<String, dynamic> json) {
    orderId = json['order_id'];
    status = json['status'];
  }

  Map<String, dynamic> toJson() {
    final Map<String, dynamic> data = new Map<String, dynamic>();
    data['order_id'] = this.orderId;
    data['status'] = this.status;
    return data;
  }
}
```

### 定義方法

前面把所需要的 `input, output` 都定義好了
那這邊就可以簡單的來呼叫看看

```dart
import 'dart:async';
import 'dart:convert';

import 'package:http/http.dart' as http;

Future<TradeResponse> buyFuture(OrderRequest order) async {
  final response = await http.put(
    Uri.parse('http://127.0.0.1:8080/trade/future/buy'),
    body: jsonEncode(order.toJson()),
  );
  if (response.statusCode == 200) {
    return TradeResponse.fromJson(jsonDecode(response.body) as Map<String, dynamic>);
  }
  throw Exception('Failed to buy future');
}
```

用了官方的 `http library`
記得還要 `import 'dart:convert';` 來做 `Dart` 裡面的 `JSON` 轉換

### 測試

因為我們還沒加入按鈕
這邊就簡單的把這個方法放入 `initState`
利用畫面重新渲染的這刻，讓他自動呼叫看看

```dart
@override
void initState() {
  initialWS();
  buyFuture(OrderRequest(code: 'MXFJ3', baseOrder: BaseOrder(price: 17777, quantity: 1)));
  super.initState();
}
```

放進去就可以重整看看
再去看看券商網站

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-27-完成下單介面『二』-03.png)

那裡面的 `17777` 是刻意讓我們比較好找的

## 總結

到這邊就算是順利的讓資料暢通
只是差一個可以點擊的按鈕
接下來的三天
就可以把畫面排一排
試試看用自己的手指按下買進、賣出
