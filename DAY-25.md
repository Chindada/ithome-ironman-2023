# 【鐵人賽】DAY-25 加入交易功能『四』

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-22-加入交易功能-03.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1247/)

## 前言

昨天把 `RESTful API` 開出來
也用 `Postman` 做了初步的確認
今天要來把核心的 `Usecase`
做一點完善
沒有意外的話，後面的五天
會全部都在 `Flutter` 介面上

## Usecase

```go
func (t *TradeUseCase) BuyFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error) {
 t.logger.Warn("BuyFuture...")
 return "", entity.StatusUnknow, nil
}

func (t *TradeUseCase) SellFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error) {
 t.logger.Warn("SellFuture...")
 return "", entity.StatusUnknow, nil
}

func (t *TradeUseCase) CancelFutureOrderByID(orderID string) (string, entity.OrderStatus, error) {
 t.logger.Warnf("CancelFutureOrderByID...%s", orderID)
 return "", entity.StatusUnknow, nil
}
```

上面三個方法
是昨天刻意留著還沒完成的
目的是快速確認後面的呼叫是暢通的
今天來把它完成

### BuyFuture

```go
func (t *TradeUseCase) BuyFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error) {
 if order.Code == "" {
  return "", entity.StatusUnknow, errors.New("empty code")
 }

 result, err := t.trade.BuyFuture(context.Background(), &pb.FutureOrderDetail{
  Code:     order.Code,
  Price:    order.Price,
  Quantity: order.Quantity,
 })
 if err != nil {
  return "", entity.StatusUnknow, err
 }

 if e := result.GetError(); e != "" {
  return "", entity.StatusUnknow, errors.New(e)
 }

 return result.GetOrderId(), entity.StringToOrderStatus(result.GetStatus()), nil
}
```

### SellFuture

```go
func (t *TradeUseCase) SellFuture(order *entity.FutureOrder) (string, entity.OrderStatus, error) {
 if order.Code == "" {
  return "", entity.StatusUnknow, errors.New("empty code")
 }

 result, err := t.trade.SellFuture(context.Background(), &pb.FutureOrderDetail{
  Code:     order.Code,
  Price:    order.Price,
  Quantity: order.Quantity,
 })
 if err != nil {
  return "", entity.StatusUnknow, err
 }

 if e := result.GetError(); e != "" {
  return "", entity.StatusUnknow, errors.New(e)
 }

 return result.GetOrderId(), entity.StringToOrderStatus(result.GetStatus()), nil
}
```

### CancelFutureOrderByID

```go
func (t *TradeUseCase) CancelFutureOrderByID(orderID string) (string, entity.OrderStatus, error) {
 result, err := t.trade.CancelFuture(context.Background(), &pb.FutureOrderID{
  OrderId: orderID,
 })
 if err != nil {
  return "", entity.StatusUnknow, err
 }

 if e := result.GetError(); e != "" {
  return "", entity.StatusUnknow, errors.New(e)
 }

 return result.GetOrderId(), entity.StringToOrderStatus(result.GetStatus()), nil
}
```

還記得結構體裡面的 `trade` 就是我們在 `New` 這個 `Object` 所建立的 `gRPC Client` 嗎
現在把他包起來
是不是整體看起來比較舒服了

## RESTful API

再來就是 `API` 接收的參數
昨天還沒有定義好，要怎麼讓使用者傳入參數
只有定義好方法是 `PUT`

我只能說是通常在這邊
大部分選擇都會讓使用者傳入一個 `JSON Body`
讓我們來把昨天開的 `Route` 完善一下

### Buy

```go
tradeRoute.PUT("/future/buy", func(c *gin.Context) {
 var order *entity.FutureOrder
 if err := c.ShouldBindJSON(&order); err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  return
 }

 orderID, status, err := tradeUsecase.BuyFuture(order)
 if err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  return
 }
 c.JSON(http.StatusOK, gin.H{
  "order_id": orderID,
  "status":   status,
 })
})
```

這邊定義好，使用者需要送出 `entity.FutureOrder` 的欄位
但使用者其實不用全部都填滿
本來應該要在這個系列也加入 `Swagge`r
但時間上應該來不及
這邊我們就先自己知道，其實只需要送『`Code`』『`Price`』『`Quantity`』

### Sell

```go
tradeRoute.PUT("/future/sell", func(c *gin.Context) {
 var order *entity.FutureOrder
 if err := c.ShouldBindJSON(&order); err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  return
 }

 orderID, status, err := tradeUsecase.SellFuture(order)
 if err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  return
 }
 c.JSON(http.StatusOK, gin.H{
  "order_id": orderID,
  "status":   status,
 })
})
```

這邊跟 Buy 只差一個字
通常會讓 `Code` 的重用性更高，但我這邊示範就先簡化了
各位可以再自行美化一番

### Cancel

```go
tradeRoute.PUT("/future/cancel", func(c *gin.Context) {
 type cancelOrder struct {
  OrderID string `json:"order_id"`
 }
 var order cancelOrder
 if err := c.ShouldBindJSON(&order); err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  return
 }
 orderID, status, err := tradeUsecase.CancelFutureOrderByID(order.OrderID)
 if err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
  return
 }
 c.JSON(http.StatusOK, gin.H{
  "order_id": orderID,
  "status":   status,
 })
})
```

取消這邊，也是差不多的意思
只是傳入值變得更簡單

## 測試

前面就是整個後端的最後一步
現在整體來跑跑看

### 買

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-25-加入交易功能『四』-01.png)

### 賣

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-25-加入交易功能『四』-02.png)

### 成果

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-25-加入交易功能『四』-03.png)

## 總結

總算完成到這步
我們現在做到從券商的 `Python API`
轉換了一層變成 `gRPC`
最後透過 `Golang` 變成 `RESTful API`
即時價格的部分更是透過 `WebSocket` 直接傳送了 `Protobuf`
到客戶端這邊，達成了高效能
明天開始就會專注在 `Flutter` 這邊
完成一個友好的下單介面
