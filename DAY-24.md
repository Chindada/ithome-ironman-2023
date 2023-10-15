# 【鐵人賽】DAY-24 加入交易功能『三』

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-22-加入交易功能-03.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1240/)

## 前言

昨天把 `Golang` 呼叫 `Python` 交易的 `gRPC` 這段串通了
雖然在取消下單這邊有一點不完美
但理念大致是對的
今天要來把這些交易功能
在 `Golang` 上實作成 `RESTful API`

## Usecase

我們在前面的天數
把各種功能包裝再包裝
最終還是要拿來使用
在 `Clean Code` 的概念裡
除了每一個被包裝的功能最好都是能夠被清楚表達意圖之外
最後的每一個商業邏輯
都是透過一個又一個的包裝功能組合
而不要在過度的加工
甚至可以說，最終的商業邏輯是零加工
僅是像堆積木一樣
把功能實作出來

對這次的專案來說
`Flutter` 是介面的部分
`Python + Golang` 是後端
很明顯是 `Flutter` 去使用後端的 `Usecase`

在實作 `RESTful API` 之前
我先把一些必須要的 `Usecase` 示範出來
可以依照需求再增加

### 從 entity 開始

這邊以期貨為例

```go
type FutureOrder struct {
 BaseOrder `json:"base_order"`

 Code   string  `json:"code"`
 Future *Future `json:"future"`
}

type BaseOrder struct {
 OrderID   string      `json:"order_id"`
 Status    OrderStatus `json:"status"`
 OrderTime time.Time   `json:"order_time"`
 Action    OrderAction `json:"action"`
 Price     float64     `json:"price"`
 Quantity  int64       `json:"quantity"`
}

const (
 // StatusUnknow -.
 StatusUnknow OrderStatus = iota
 // StatusPendingSubmit -.
 StatusPendingSubmit
 // StatusPreSubmitted -.
 StatusPreSubmitted
 // StatusSubmitted -.
 StatusSubmitted
 // StatusFailed -.
 StatusFailed
 // StatusCancelled -.
 StatusCancelled
 // StatusFilled -.
 StatusFilled
 // StatusPartFilled -.
 StatusPartFilled
)

type OrderStatus int64

func (s OrderStatus) String() string {
 switch s {
 case StatusUnknow:
  return "Unknow"
 case StatusPendingSubmit:
  return "PendingSubmit"
 case StatusPreSubmitted:
  return "PreSubmitted"
 case StatusSubmitted:
  return "Submitted"
 case StatusFailed:
  return "Failed"
 case StatusCancelled:
  return "Cancelled"
 case StatusFilled:
  return "Filled"
 case StatusPartFilled:
  return "PartFilled"
 default:
  return ""
 }
}

const (
 // ActionNone -.
 ActionNone OrderAction = iota
 // ActionBuy -.
 ActionBuy
 // ActionSell -.
 ActionSell
)

type OrderAction int64

func (a OrderAction) String() string {
 switch a {
 case ActionNone:
  return ActionStringNone
 case ActionBuy:
  return ActionStringBuy
 case ActionSell:
  return ActionStringSell
 default:
  return ""
 }
}

type Future struct {
 Code           string    `json:"code"`
 Symbol         string    `json:"symbol"`
 Name           string    `json:"name"`
 Category       string    `json:"category"`
 DeliveryMonth  string    `json:"delivery_month"`
 DeliveryDate   time.Time `json:"delivery_date"`
 UnderlyingKind string    `json:"underlying_kind"`
 Unit           int64     `json:"unit"`
 LimitUp        float64   `json:"limit_up"`
 LimitDown      float64   `json:"limit_down"`
 Reference      float64   `json:"reference"`
 UpdateDate     time.Time `json:"update_date"`
}
```

這邊就偷懶不講解太細節的東西
基本上就是一張單，該有的欄位
股票或者期貨都是差不多
這次的鐵人賽會比較多篇幅在期貨
其中一個原因就是寫稿的時間，大部分股票都已經收盤🤣

### Interface

```go
type Trade interface {
 BuyFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error)
 SellFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error)
 CancelFutureOrderByID(orderID string) (string, entity.OrderStatus, error)
}
```

這邊定義了三個方法
是交易期貨最基本的三個方法『買』『賣』『取消』
先來定義一下我們需要的 `struct`

```go
type TradeUseCase struct {
 trade  pb.TradeInterfaceClient
 logger *log.Log
}
```

首先，至少需要一個跟 `Python` 那邊的 `gRPC` 接口
然後就可以很快完成下面的雛形

```go
func NewTrade() Trade {
 logger := log.Get()
 conn, err := grpc.Dial(address, grpc.WithInsecure())
 if err != nil {
  logger.Fatalf("did not connect: %v", err)
 }
 trade := pb.NewTradeInterfaceClient(conn)
 return &TradeUseCase{
  trade:  trade,
  logger: logger,
 }
}

func (t *TradeUseCase) BuyFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error) {
 t.logger.Warn("BuyFuture...")
 return "", entity.StatusUnknow, nil
}

func (t *TradeUseCase) SellFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error) {
 t.logger.Warn("SellFuture...")
 return "", entity.StatusUnknow, nil
}

func (t *TradeUseCase) CancelFutureOrderByID(orderID string) (string, entity.OrderStatus, error) {
 t.logger.Warn("CancelFutureOrderByID...")
 return "", entity.StatusUnknow, nil
}
```

這邊只是雛型
先讓我們快速的往下寫

## Gin@Golang

要建立 `RESTful API`
會繼續沿用之前的 [Gin 框架](https://gin-gonic.com)
剛剛完成了交易相關的 `Usecase`
在 `API` 這層，就是簡單地拿來使用

先來開一個 `Route`

```go
tradeUsecase := trade.NewTrade()
tradeRoute := handler.Group("/trade")
tradeRoute.PUT("/future/buy", func(c *gin.Context) {
 tradeUsecase.BuyFuture(nil)
 c.JSON(http.StatusOK, nil)
})
tradeRoute.PUT("/future/sell", func(c *gin.Context) {
 tradeUsecase.SellFuture(nil)
 c.JSON(http.StatusOK, nil)
})
tradeRoute.PUT("/future/cancel", func(c *gin.Context) {
 type cancelOrder struct {
  OrderID string `json:"order_id"`
 }
 var order cancelOrder
 if err := c.ShouldBindJSON(&order); err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  return
 }
 tradeUsecase.CancelFutureOrderByID(order.OrderID)
 c.JSON(http.StatusOK, nil)
})
```

先簡單的確認 `API` 是否有成功開啟，並能夠被呼叫
如果在 `debug mode` 下 `gin` 應該要顯示出以下路由

```bash
[GIN-debug] PUT    /trade/future/buy         --> main.main.func2 (2 handlers)
[GIN-debug] PUT    /trade/future/sell        --> main.main.func3 (2 handlers)
[GIN-debug] PUT    /trade/future/cancel      --> main.main.func4 (2 handlers)
```

## Postman

用 Postman 來試試看

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-24-加入交易功能『三』-01.png)

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-24-加入交易功能『三』-02.png)

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-24-加入交易功能『三』-03.png)

這時候回來看一下 `Golang` 印了什麼

```bash
WARN[2023-10-09 10:19:40] BuyFuture...
WARN[2023-10-09 10:20:31] SellFuture...
WARN[2023-10-09 10:21:22] CancelFutureOrderByID...TEST_ID
```

## 總結

很順利的把 `RESTful API`
簡單的開出來
但別忘記，我們在 `Usecase` 裡面還沒有把真正的交易放進去
明天將會是後端的完成日
放進去，就可以來好好地刻 `Flutter` 介面
