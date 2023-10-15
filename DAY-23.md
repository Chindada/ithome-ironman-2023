# 【鐵人賽】DAY-23 加入交易功能『二』

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-22-加入交易功能-03.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1234/)

## 前言

昨天完成的 `Python` 交易功能
然後也定義了 `gRPC` 的介面
今天會來把 `Golang` 呼叫 `Python` 這段也建立起來
但其實就跟之前的功能其實挺像
那我們就來開始

## Python

幫各位複習一下
實作 `gRPC` 介面，本身是一件抽象的事情
只要實作出跟 `*.proto` 當中的 `rpc`
相同的命名的方法
只要能被呼叫，有正確的回傳都是可以的
所以在前些日子有提到
日後有第二第三家券商提供 `API`
我們一樣可以將它包裝成一樣的 `rpc`
進入這個系統當中
當然，這也考驗著最一開始的系統設計

### 包裝

```python
from pb import (
    basic_pb2,
    basic_pb2_grpc,
    entity_pb2,
    history_pb2,
    history_pb2_grpc,
    realtime_pb2,
    realtime_pb2_grpc,
    subscribe_pb2,
    subscribe_pb2_grpc,
    trade_pb2,
    trade_pb2_grpc,
)

class RPCTrade(trade_pb2_grpc.TradeInterfaceServicer):
    def __init__(self, shioaji: Shioaji):
        self.shioaji = shioaji
        self.rabbit = shioaji.rabbit
```

定義了一個叫做 `RPCTrade` 的 `class`
需要傳入

- `Shioaji API`

- `RabbitMQ` 的連線

### 股票

```python
def BuyStock(self, request, _):
    result = self.shioaji.buy_stock(
        request.stock_num,
        request.price,
        request.quantity,
    )
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )

def SellStock(self, request, _):
    result = self.shioaji.sell_stock(
        request.stock_num,
        request.price,
        request.quantity,
    )
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )

def SellFirstStock(self, request, _):
    result = self.shioaji.sell_first_stock(
        request.stock_num,
        request.price,
        request.quantity,
    )
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )

def CancelStock(self, request, _):
    result = self.shioaji.cancel_stock(request.order_id)
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )
```

### 期貨

```python
def BuyFuture(self, request, _):
    result = self.shioaji.buy_future(
        request.code,
        request.price,
        request.quantity,
    )
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )

def SellFuture(self, request, _):
    result = self.shioaji.sell_future(
        request.code,
        request.price,
        request.quantity,
    )
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )

def SellFirstFuture(self, request, _):
    result = self.shioaji.sell_first_future(
        request.code,
        request.price,
        request.quantity,
    )
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )

def CancelFuture(self, request, _):
    result = self.shioaji.cancel_future(
        request.order_id,
    )
    return trade_pb2.TradeResult(
        order_id=result.order_id,
        status=result.status,
        error=result.error,
    )
```

以上兩種包裝起來
基本大同小異
那該如何加入到 `Python` 這邊的 `gRPC Server` 呢

## gRPC Server

```python
def run_server():
    server = grpc.server(futures.ThreadPoolExecutor())
    basic_pb2_grpc.add_BasicDataInterfaceServicer_to_server(BASIC, server)
    history_pb2_grpc.add_HistoryDataInterfaceServicer_to_server(RPCHistory(BASIC.get_shioaji()), server)
    trade_pb2_grpc.add_TradeInterfaceServicer_to_server(RPCTrade(BASIC.get_shioaji()), server)

    server.add_insecure_port("[::]:50051")
    print("Server started. Listening on port 50051.")
    server.start()
    server.wait_for_termination()
```

依樣畫葫蘆
前面怎麼加就怎麼加
這邊只是快速複習一下

## Golang

該輪到 Golang 來呼叫剛剛辛辛苦苦加入的 gRPC 方法

```go
func main() {
 conn, err := grpc.Dial(address, grpc.WithInsecure())
 if err != nil {
  logger.Fatalf("did not connect: %v", err)
 }
 defer conn.Close()
 c := pb.NewBasicDataInterfaceClient(conn)

 _, err = c.Login(context.Background(), &emptypb.Empty{})
 if err != nil {
  logger.Fatalf("could not greet: %v", err)
 }

 trade := pb.NewTradeInterfaceClient(conn)
 result, err := trade.BuyStock(context.Background(), &pb.StockOrderDetail{
  StockNum: "2317",
  Price:    105,
  Quantity: 1,
 })
 if err != nil {
  logger.Fatalf("BuyStock not greet: %v", err)
 } else {
  logger.Infof("BuyStock result: %v", result)
 }
}
```

先簡單的直接跑一下
然後我們試試看先 `Hard Code` 一張『鴻海-2317』價格 105
跑起來應該會像下面這樣

```bash
INFO[2023-10-08 21:56:27] BuyStock result: order_id:"14a572d3" status:"PendingSubmit"
```

看起來是沒有錯誤
成功獲得了一個 `order_id`
但最重要的還是要去看一下
我們在券商那邊的系統
是否有真的下到這張

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-23-加入交易功能『二』-01.png)

有了哦！
那我們現在來試試看
取消這張單
還記得昨天提到的
對券商來說，只認 `order_id`
無論這張單是『股票』『期貨』的買入、賣出
來試一下，買完直接取消

```go
func main() {
 conn, err := grpc.Dial(address, grpc.WithInsecure())
 if err != nil {
  logger.Fatalf("did not connect: %v", err)
 }
 defer conn.Close()
 c := pb.NewBasicDataInterfaceClient(conn)

 _, err = c.Login(context.Background(), &emptypb.Empty{})
 if err != nil {
  logger.Fatalf("could not greet: %v", err)
 }

 trade := pb.NewTradeInterfaceClient(conn)
 buyResult, buyErr := trade.BuyStock(context.Background(), &pb.StockOrderDetail{
  StockNum: "2317",
  Price:    105,
  Quantity: 1,
 })
 if buyErr != nil {
  logger.Fatalf("BuyStock not greet: %v", buyErr)
 } else {
  logger.Infof("BuyStock result: %v", buyResult)
 }

 cancelResult, cancelErr := trade.CancelStock(context.Background(), &pb.OrderID{
  OrderId: "",
 })
 if err != nil {
  logger.Fatalf("CancelStock not greet: %v", cancelErr)
 } else {
  logger.Infof("CancelStock result: %v", cancelResult)
 }
}
```

沒意外的話，應該失敗了吧

```bash
INFO[2023-10-08 22:06:50] BuyStock result: order_id:"ae1d27a6" status:"PendingSubmit"
INFO[2023-10-08 22:06:50] CancelStock result: error:"id not found"
```

被提示找不到 `order_id`
因為這邊漏了去更新我們內部的資料
這邊算是針對永豐這邊的特別處理
也許不是所有券商都需要這樣做
現在要做的就是再有新的單狀態變化的時候
自動更新內存

### 回到 Python 加一下

這邊需要加入的是叫做 `order_callback`
具體來說就是有單變化，就會自動更新內部

```python
def update_local_order(self):
    with self.__order_map_lock:
        cache = self.__order_map.copy()
        self.__order_map = {}
        try:
            self.__api.update_status()
            for order in self.__api.list_trades():
                self.__order_map[order.order.id] = order
        except Exception:
            self.__order_map = cache
```

小小解釋一下
這邊用了一個鎖
保證不會有更新錯誤
使用了券商提供的 `[update_status()](https://sinotrade.github.io/tutor/order/UpdateStatus/)`
然後再放到我們 `cache`
要改在登入的地方，如下

```python
def login(self, user: ShioajiAuth):
    def login_cb(security_type: sc.SecurityType):
        with self.__login_status_lock:
            if security_type.value in [item.value for item in sc.SecurityType]:
                self.__login_progess += 1
                logger.info("login progress: %d/4, %s", self.__login_progess, security_type)

    try:
        self.__api.login(
            api_key=user.api_key,
            secret_key=user.api_key_secret,
            contracts_cb=login_cb,
            subscribe_trade=True,
        )

    except SystemMaintenance as exc:
        time.sleep(75)
        raise RuntimeError("login 503 system maintenance") from exc

    except Exception as error:
        time.sleep(30)
        raise RuntimeError("login error") from error

    while True:
        with self.__login_status_lock:
            if self.__login_progess == 4:
                break

    self.__api.activate_ca(
        ca_path=f"./data/{user.person_id}.pfx",
        ca_passwd=user.ca_password,
        person_id=user.person_id,
    )
    self.fill_stock_map()
    self.fill_future_map()
    self.__api.set_order_callback(self.order_callback)
    return self

def order_callback(self, order_state: sc.OrderState, res: dict):
    self.update_local_order()
```

### 回到 Golang 試試

如果你跟筆者一樣
是在非開盤時間寫這段
大概還是會失敗

## 總結

今天算是串好一半
取消這段，要等到開盤時間
再來試試
明天我們來把這些交易都開成 `API`
最後六天就會專心地把這些功能整一整
變成一個介面
