# 【鐵人賽】DAY-09 進入 Golang 『一』

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-09-進入-Golang-『一』-02.jpg)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/895/)

## 前言

雖然前面的 Python 券商明顯還沒完成
很容易就知道至少還缺下單、串流等等
但這次專案有三大區塊：Python, Golang, Flutter
其實缺哪塊都不算完成
今天決定先來進入 Golang🤓
然後一路進到 `Flutter`
完成一條路線的功能，先獲得成就感！！

## 筆者環境

這邊我就跳過環境的建置了
多講可能到 30 天，還沒半個畫面出現🤣

- Golang: 1.21.1

- Protoc: libprotoc 24.3

- protoc-gen-go-grpc: 1.3.0

- protoc-gen-go: v1.31.0

以上為我目前使用的環境
這邊也沒辦法很肯定地說，多低的版本一定不行
因為我通常都會一直頻繁的更新
畢竟不是公司的專案，看到有新的就會手癢去更新

## 專案結構

`Clean Code`
我想沒有多少人可以說自己的程式碼是這個境界
但我想這應該是大部分人的目標
我這邊推薦一個 Github 上 Golang Star 比較多的 Clean-Arch

- [go-clean-arch](https://github.com/bxcodec/go-clean-arch#go-clean-arch)

- [Go Clean template](https://github.com/evrone/go-clean-template#go-clean-template)

Clean Code 我就不敢妄言教學
但只要能夠以這個方向去開發
我相信不會過一段時間就難以擴展功能

```bash
# 下面提供給各位參考
.
├── cmd
│   └── app
├── configs
├── data
├── docs
│   ├── docs.go
│   ├── swagger.json
│   └── swagger.yaml
├── go.mod
├── go.sum
├── internal
│   ├── app
│   ├── controller
│   ├── entity
│   ├── modules
│   ├── usecase
│   └── utils
├── logs
├── migrations
├── pb
├── pkg
│   ├── cache
│   ├── eventbus
│   ├── grpc
│   ├── httpserver
│   ├── log
│   ├── postgres
│   └── rabbitmq
├── scripts
└── toc-machine-trading
```

## 開出第一隻 API（使用 [Gin](https://github.com/gin-gonic/gin)）

到這邊其實會有很多人有疑問
在 Python 那邊也可以開 API
為什麼還要特別弄一個 Golang
第一個當然是效能
再來就是比說我們在 Python 那邊已經取得股票的基本資料
以筆者的經驗來說 Golang 在處理這段效能還是比較快的
後續更多的商業邏輯出現
高併發一直是 Golang 的特點（當然不能夠無腦一直併發）

### 將股票加入 Protobuf

```c
// StockDetailMessage is the message for stock detail
message StockDetailMessage {
    string exchange    = 1;
    string category    = 2;
    string code        = 3;
    string name        = 4;
    double reference   = 5;
    string update_date = 6;
    string day_trade   = 7;
}

// StockDetailResponse is the response for stock detail
message StockDetailResponse {
    repeated StockDetailMessage stock = 1;
}

// BasicDataInterface is the interface for basic data service
service BasicDataInterface {
    // GetAllStockDetail is the function to get stock detail
    rpc GetAllStockDetail(google.protobuf.Empty) returns (StockDetailResponse) {}
}
```

這段就直接一次寫在一起
定義了基本資料的格式以及取得的方法

```go
package main

import (
 "context"
 "io"
 "log"
 "net"
 "net/http"
 "time"

 "gop/pb"

 "github.com/gin-gonic/gin"
 "google.golang.org/grpc"
 "google.golang.org/protobuf/types/known/emptypb"
)

const (
 address                   = "localhost:50051"
 _defaultReadTimeout       = 5 * time.Second
 _defaultReadHeaderTimeout = 5 * time.Second
 _defaultWriteTimeout      = 5 * time.Minute
)

func main() {
 conn, err := grpc.Dial(address, grpc.WithInsecure())
 if err != nil {
  log.Fatalf("did not connect: %v", err)
 }
 defer conn.Close()
 c := pb.NewBasicDataInterfaceClient(conn)

 _, err = c.Login(context.Background(), &emptypb.Empty{})
 if err != nil {
  log.Fatalf("could not greet: %v", err)
 }

 stockArr, err := c.GetAllStockDetail(context.Background(), &emptypb.Empty{})
 if err != nil {
  log.Fatalf("could not greet: %v", err)
 }

 handler := gin.New()
 handler.Use(gin.Recovery())
 basicRoute := handler.Group("/basic-data")
 basicRoute.GET("/stock-data", func(c *gin.Context) {
  c.JSON(http.StatusOK, gin.H{
   "data": stockArr,
  })
 })

 srv := &http.Server{
  ErrorLog:          log.New(io.Discard, "", 0),
  Handler:           handler,
  Addr:              net.JoinHostPort("", "8080"),
  ReadHeaderTimeout: _defaultReadHeaderTimeout,
  ReadTimeout:       _defaultReadTimeout,
  WriteTimeout:      _defaultWriteTimeout,
 }
 srv.ListenAndServe()
}
```

今天先簡單示範一下 gin 開出來是什麼感覺
跑起來之後，就可以試試用 Postman 之類的工具要要看

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-09-進入-Golang-『一』-01.png)

這樣這隻 RESTful API 就是可以給 Flutter 來呼叫

## 總結

今天文中雖然有提到 Clean Code
但示範上並沒有按照正規的方式來
主要還是想快速呈現
想要完整的呈現在一篇網誌中，還是比較困難
之後可能還是會繼續像這樣篇 Prototype 的形式
