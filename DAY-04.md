# 【鐵人賽】DAY-04 初探 protobuf、gRPC 『二』

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-07.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/797/)

## 前言

昨天入門了 `Protobuf`
有提到這是作為 `Process` 之間『溝通』用的『訊息結構』
那今天理所當然的要進入到溝通這個行為所建立在的 `**gRPC**`
過程中，會直接用本次整個專案的框架來演示
就是會以跨語言的方式來實作（`Python`, `Golang`）

## 為什麼要用 gRPC

1. **高效性**： gRPC 使用Protobuf 而非 XML 或 JSON，因此較節省頻寬，並減少序列化和反序列化的時間與資料量

3. **多語言支援**： gRPC 支援多種程式語言，使得不同平台和系統能夠通過相同的API進行通信

5. **自動生成程式碼**： 使用 Protobuf 定義介面後，gRPC 工具可以自動生成用於客戶端和伺服器的程式碼

7. **長連線**： 允許雙向通信，這對於需要實時更新的應用非常有用

## 擴展 Proto

這邊為什麼用到『擴展』呢
昨天我們定義了一個只有期貨單的 Proto

```c
syntax            = "proto3";
option go_package = "./pb";
package toc_python_forwarder;

message FutureOrderDetail {
    string code    = 1;
    double price   = 2;
    int64 quantity = 3;
}
```

到這邊，只有訊息結構
但還沒有定義『溝通』的方法
讓我們來稍微完善一下
初步定義一個購買期貨的方法
並且加入一個訊息結構來回傳下單的結果

```c
service TradeInterface {
    rpc BuyFuture(FutureOrderDetail) returns (TradeResult) {}
}

message TradeResult {
    string order_id = 1;
    string status   = 2;
    string error    = 3;
}
```

讓我們把他們放在一起

```c
syntax            = "proto3";
option go_package = "./pb";
package toc_python_forwarder;

service TradeInterface {
    rpc BuyFuture(FutureOrderDetail) returns (TradeResult) {}
}

message FutureOrderDetail {
    string code    = 1;
    double price   = 2;
    int64 quantity = 3;
}

message TradeResult {
    string order_id = 1;
    string status   = 2;
    string error    = 3;
}
```

## 編譯

這部分在昨天的 Protobuf 中
我已經不小心寫上 gRPC 在編譯所需要的參數
但還是在演示一次

### Python

```bash
$pip3 install -U \
  --no-warn-script-location \
  --no-cache-dir \
  grpcio \
  grpcio-tools

$outpath=./src/pb
$python3 -m grpc_tools.protoc \
  --python_out=$outpath \
  --grpc_python_out=$outpath \
  --mypy_out=$outpath \
  --proto_path=./trade-protobuf/protos/v3/app \
  --proto_path=./trade-protobuf/protos/v3/forwarder \
  ./trade-protobuf/protos/v3/*/*.proto
```

### Golang

```bash
protoc \
    --go_out=. \
    --go-grpc_out=. \
    --proto_path=./trade-protobuf/protos/v3/app \
    --proto_path=./trade-protobuf/protos/v3/forwarder \
    ./trade-protobuf/protos/v3/*/*.proto
```

因為 Flutter 那邊就真的單純是直接拿來取代 `JSON`
並沒有任何地方會使用到 `gRPC`
所以這邊並沒有講述到 `Flutter gRPC` 的編譯

## Python Server 端

前面完成了編譯
我們來試著起起看 `gRPC Server`
因為只是展示，回傳的資料也是先做一下假的～

```python
# Import the necessary gRPC libraries
from concurrent import futures

import grpc

# Import the generated gRPC files
from pb import trade_pb2, trade_pb2_grpc


# Define the gRPC service
class Trader(trade_pb2_grpc.TradeInterfaceServicer):
    def BuyFuture(self, request, context):
        return trade_pb2.TradeResult(
            order_id="123456",
            status="success",
            error="",
        )


# Run the gRPC server
def run_server():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    trade_pb2_grpc.add_TradeInterfaceServicer_to_server(Trader(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    print("Server started. Listening on port 50051.")
    server.wait_for_termination()


if __name__ == "__main__":
    run_server()
```

## Golang Client 端

```go
package main

import (
 "context"
 "fmt"
 "log"

 "gop/pb"

 "google.golang.org/grpc"
)

const (
 address = "localhost:50051"
)

func main() {
 conn, err := grpc.Dial(address, grpc.WithInsecure())
 if err != nil {
  log.Fatalf("did not connect: %v", err)
 }
 defer conn.Close()
 c := pb.NewTradeInterfaceClient(conn)

 r, err := c.BuyFuture(context.Background(), &pb.FutureOrderDetail{
  Code:     "IRONMAN DEMO",
  Price:    100,
  Quantity: 1,
 })
 if err != nil {
  log.Fatalf("could not greet: %v", err)
 }

 fmt.Printf("Response from server: %s\n", r.GetStatus())
}
```

## 整合

**Server**

```bash
$PBPATH=$(PWD)/src/pb
$PYTHONPATH=$(PBPATH) python3 -BO ./src/main.py

Server started. Listening on port 50051.
BuyFuture called, code is IRONMAN DEMO
```

**Client**

```bash
$go run main.go

Response from server: success
```

## 總結

到這邊就算是初步的展示了一下 gRPC 在跨 Process 的溝通過程
接下來就要準備進入到核心的商業邏輯
