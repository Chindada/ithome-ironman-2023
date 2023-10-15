# 【鐵人賽】DAY-22 加入交易功能

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-22-加入交易功能-03.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1171/)

## 前言

剛好花了三週把三種程式語言走了一遍
完成了所有的技術債
對於一個自己的下單介面
要進入到最核心的功能
就是交易，包含『買進』『賣出』
最後這幾天會快速地走一遍從券商到介面這段

## 文件

一樣，先來看一下[券商的文件](https://sinotrade.github.io/tutor/order/Stock/)

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-22-加入交易功能-01.png)

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-22-加入交易功能-02.png)(#place_order_doc)

## 包裝

可以看得出來
在最源頭的 `API` 是沒有分買進或賣出
而是區分 `Order`
而 `Order` 又可以再進一步分成『買進』『賣出』『先賣』
在這三個大分類之上，還可以利用 `Contract` 在區分成『股票』『期貨』
在 `Order` 當中也有一些細節是針對股票或者期貨有不同的用法
券商把 `API` 設計的這麼活
當然就是讓我們再包裝一次
我想應該沒有人想要每次下單
都要再組一次這麼多東西
所以這邊會把『股票』『期貨』的『買進』『賣出』『先賣』
再次包裝成 `2*3=6` 個方法

### 買進股票

```python
def buy_stock(self, stock_num: str, price: float, quantity: int):
    order: Order = self.__api.Order(
        price=price,
        quantity=quantity,
        action=sc.Action.Buy,
        price_type=sc.StockPriceType.LMT,
        order_type=sc.OrderType.ROD,
        order_lot=sc.StockOrderLot.Common,
        account=self.__api.stock_account,
    )
    trade = self.__api.place_order(self.get_contract_by_stock_num(stock_num), order)
    if trade is not None and trade.order.id != "":
        return OrderStatus(trade.order.id, trade.status.status, "")
    return OrderStatus("", "", "buy stock fail")
```

### 賣出股票

```python
def sell_stock(self, stock_num: str, price: float, quantity: int):
    order: Order = self.__api.Order(
        price=price,
        quantity=quantity,
        action=sc.Action.Sell,
        price_type=sc.StockPriceType.LMT,
        order_type=sc.OrderType.ROD,
        order_lot=sc.StockOrderLot.Common,
        account=self.__api.stock_account,
    )
    trade = self.__api.place_order(self.get_contract_by_stock_num(stock_num), order)
    if trade is not None and trade.order.id != "":
        return OrderStatus(trade.order.id, trade.status.status, "")
    return OrderStatus("", "", "sell stock fail")
```

### 先賣股票

```python
def sell_first_stock(self, stock_num: str, price: float, quantity: int):
    order: Order = self.__api.Order(
        price=price,
        quantity=quantity,
        action=sc.Action.Sell,
        price_type=sc.StockPriceType.LMT,
        order_type=sc.OrderType.ROD,
        order_lot=sc.StockOrderLot.Common,
        daytrade_short=True,
        account=self.__api.stock_account,
    )
    trade = self.__api.place_order(self.get_contract_by_stock_num(stock_num), order)
    if trade is not None and trade.order.id != "":
        return OrderStatus(trade.order.id, trade.status.status, "")
    return OrderStatus("", "", "sell first stock fail")
```

### 買進期貨

```python
def buy_future(self, code: str, price: float, quantity: int):
    order: Order = self.__api.Order(
        price=price,
        quantity=quantity,
        action=sc.Action.Buy,
        price_type=sc.FuturesPriceType.LMT,
        order_type=sc.OrderType.ROD,
        octype=sc.FuturesOCType.Auto,
        account=self.__api.futopt_account,
    )
    trade = self.__api.place_order(contract=self.get_contract_by_future_code(code), order=order)
    if trade is not None and trade.order.id != "":
        return OrderStatus(trade.order.id, trade.status.status, "")
    return OrderStatus("", "", "buy future fail")
```

### 賣出期貨

```python
def sell_future(self, code: str, price: float, quantity: int):
    order: Order = self.__api.Order(
        price=price,
        quantity=quantity,
        action=sc.Action.Sell,
        price_type=sc.FuturesPriceType.LMT,
        order_type=sc.OrderType.ROD,
        octype=sc.FuturesOCType.Auto,
        account=self.__api.futopt_account,
    )
    trade = self.__api.place_order(contract=self.get_contract_by_future_code(code), order=order)
    if trade is not None and trade.order.id != "":
        return OrderStatus(trade.order.id, trade.status.status, "")
    return OrderStatus("", "", "sell future fail")
```

### 先賣期貨

```python
def sell_first_future(self, code: str, price: float, quantity: int):
    order: Order = self.__api.Order(
        price=price,
        quantity=quantity,
        action=sc.Action.Sell,
        price_type=sc.FuturesPriceType.LMT,
        order_type=sc.OrderType.ROD,
        octype=sc.FuturesOCType.Auto,
        account=self.__api.futopt_account,
    )
    trade = self.__api.place_order(contract=self.get_contract_by_future_code(code), order=order)
    if trade is not None and trade.order.id != "":
        return OrderStatus(trade.order.id, trade.status.status, "")
    return OrderStatus("", "", "sell first future fail")
```

來解釋一下
這邊把下單簡化了一些
輸入只需要『代碼』『數量』『價格』
其餘的屬性都固定，皆是『限價』『`ROD`』
這邊只是提供一個範例
完全可以再依照自己的需求修改
想要的話，多包幾種，寫得更漂亮也是可以

但感覺好像缺少了什麼
我們一般下單，還可以取消啊
這邊來補充一下
取消下單在這邊來說
並沒有區分『股票』『期貨』
而是針對前面六種方法所產生的 `Order ID`
這邊需要再解釋一下
前面的 `Code` 當中，有多了一個 `OrderStatus`
這是我自己在新增的一個 `class`

```python
class OrderStatus:
    def __init__(self, order_id: str, status: str, error: str):
        self.order_id = order_id
        self.status = status
        self.error = error
```

把[文件](#place_order_doc)中 `Out(Response)` 取出

- `order_id`

- `status`

另外的 `error` 是嵌套進去的錯誤
可以後續作為判斷下單的錯誤

## gRPC

前面做完 `Python` 串券商的部分
這次的[系列](https://tocandraw.com/category/2023-ironman/)可不只是到這邊
而是要再次把這邊抽象包裝成 `gRPC`
來看一下我定義好的 `rpc`

```c
syntax            = "proto3";
option go_package = "./pb";
package toc_python_forwarder;

import "google/protobuf/empty.proto";
import "entity.proto";

service TradeInterface {
    rpc BuyStock(StockOrderDetail) returns (TradeResult) {}
    rpc SellStock(StockOrderDetail) returns (TradeResult) {}
    rpc SellFirstStock(StockOrderDetail) returns (TradeResult) {}
    rpc CancelStock(OrderID) returns (TradeResult) {}
    rpc BuyFuture(FutureOrderDetail) returns (TradeResult) {}
    rpc SellFuture(FutureOrderDetail) returns (TradeResult) {}
    rpc SellFirstFuture(FutureOrderDetail) returns (TradeResult) {}
    rpc CancelFuture(FutureOrderID) returns (TradeResult) {}
    rpc GetLocalOrderStatusArr(google.protobuf.Empty) returns (google.protobuf.Empty) {}
    rpc GetOrderStatusByID(OrderID) returns (TradeResult) {}
}
```

把剛剛的 `Python` 方法都定義成 `gRPC`
避免混淆，把前面回傳的 `OrderStatus` 在 `Protobuf` 中變成 `TradeResult`

```c
message TradeResult {
    string order_id = 1;
    string status   = 2;
    string error    = 3;
}
```

當然，也別忘了 `OrderID`

```c
message OrderID {
    string order_id = 1;
    bool simulate   = 2;
}
```

## 總結

今天先進行到這邊
明天會來把這段 gRPC 實作
並且可以順利讓 `Golang` 呼叫
