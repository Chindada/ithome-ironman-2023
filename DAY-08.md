# 【鐵人賽】DAY-08 從券商 API 開始『三』

![IMG](https://tocandraw.com/wp-content/uploads/2022/04/trade_agent_day_kbar.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/855/)

## 前言

昨天我們取得了基本資料
今天會進入到去券商那邊取得歷史資料
一個下單軟體，還是要能夠有一些資料
不然就有點盲目下單的感覺了

## 包裝

繼續重複著相同的事情
將[官方](https://sinotrade.github.io/tutor/market_data/historical/)提供的查詢歷史資料方法
包裝一層到數層不等，來符合我們的架構
最終需要變成 `gRPC` 的接口
傳輸的媒介應該要是 `Protobuf`
歷史資料有很多種類型可以選擇
今天會挑最常看到的 K 線來做演示

### K 線

先看文件

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-08-從券商-API-開始『三』-01.png)

看起來相當簡單
傳入開始日期、結束日期
以及股票號碼
就可以做第一層包裝，如下

```python
def stock_kbars(self, num, date):
    try:
        return self.__api.kbars(
            contract=self.get_contract_by_stock_num(num),
            start=date,
            end=date,
        )
    except TimeoutError:
        return self.stock_kbars(num, date)
```

可以很輕易注意到，我的 `date`，同時拿來當作 `start, end`
一次只處理一天，這蠻看人
我不太喜歡一次拿了好多天，又要再一個陣列裡面去做切割
那就可以來試跑一下
沒意外你會看到一大坨資料

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-08-從券商-API-開始『三』-02.png)

當然，這不會是我們最終需要的型態
先來定義一下 `proto`

```c
// HistoryKbarMessage
message HistoryKbarMessage {
    int64 ts     = 1;
    double close = 2;
    double open  = 3;
    double high  = 4;
    double low   = 5;
    int64 volume = 6;
    string code  = 7;
}
```

編譯過程我就省略了
可以看得出來
我把券商 API 給的資料
拆散，重新整合成傳統意義上的 OHLC

> **美國線**（英語：Open-High-Low-Close chart，**OHLC** chart），以豎立的線條表現[股票](https://zh.wikipedia.org/wiki/%E8%82%A1%E7%A5%A8)價格的變化，可以呈現「開盤價、最高價、最低價、收盤價」，豎線呈現最高價和最低價間的價差間距，左側橫線代表開盤價，右側橫線代表收盤價，繪製上較[K線](https://zh.wikipedia.org/wiki/K%E7%B7%9A)簡單。另有一種美國線僅呈現「最高價、最低價、收盤價」（**HLC**）三項訊息。
>
> 另外，在線圖的繪製上，也有人採用單色製圖，並不影響「開盤價、最高價、最低價、收盤價」四項訊息的呈現。
>
> 引述自[維基百科](https://zh.wikipedia.org/zh-tw/美國線)

至少我認為這樣是我比較好閱讀的
在包裝一次
這次加入 `gRPC`

```python
service HistoryDataInterface {
    rpc GetStockHistoryKbar(StockNumArrWithDate) returns (HistoryKbarResponse) {}
}
```

下面是節錄這次比較重要的部分

```python
class RPCHistory(history_pb2_grpc.HistoryDataInterfaceServicer):
    def __init__(self, worker: Shioaji):
        self.worker = worker

    def GetStockHistoryKbar(self, request, _):
        response = history_pb2.HistoryKbarResponse()
        threads = []

        for num in request.stock_num_arr:
            thread = threading.Thread(
                target=self.fill_stock_history_kbar_response,
                args=(
                    num,
                    request.date,
                    response,
                    self.worker,
                ),
                daemon=True,
            )
            threads.append(thread)
            thread.start()
        for thread in threads:
            thread.join()
        return response

    def fill_stock_history_kbar_response(self, num, date, response, worker: Shioaji):
        kbar = worker.stock_kbars(num, date)
        total_count = len(kbar.ts)
        tmp_length = [
            len(kbar.Close),
            len(kbar.Open),
            len(kbar.High),
            len(kbar.Low),
            len(kbar.Volume),
        ]

        for length in tmp_length:
            if length - total_count != 0:
                return

        for pos in range(total_count):
            response.data.append(
                history_pb2.HistoryKbarMessage(
                    code=num,
                    ts=kbar.ts[pos],
                    close=kbar.Close[pos],
                    open=kbar.Open[pos],
                    high=kbar.High[pos],
                    low=kbar.Low[pos],
                    volume=kbar.Volume[pos],
                )
            )


# Run the gRPC server
def run_server():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    basic = Basic()
    basic_pb2_grpc.add_BasicDataInterfaceServicer_to_server(basic, server)
    history_pb2_grpc.add_HistoryDataInterfaceServicer_to_server(RPCHistory(basic.get_shioaji()), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    print("Server started. Listening on port 50051.")
    server.wait_for_termination()
```

也用一個比較簡單的 Golang 來試著串串

```go
package main

import (
 "context"
 "fmt"
 "log"

 "gop/pb"

 "google.golang.org/grpc"
 "google.golang.org/protobuf/types/known/emptypb"
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
 c := pb.NewBasicDataInterfaceClient(conn)

 _, err = c.Login(context.Background(), &emptypb.Empty{})
 if err != nil {
  log.Fatalf("could not greet: %v", err)
 }

 h := pb.NewHistoryDataInterfaceClient(conn)
 kbarArr, err := h.GetStockHistoryKbar(context.Background(), &pb.StockNumArrWithDate{
  StockNumArr: []string{"2330"},
  Date:        "2023-09-22",
 })
 if err != nil {
  log.Fatalf("could not greet: %v", err)
 }

 for _, v := range kbarArr.GetData() {
  fmt.Println(v)
 }
}
```

可以看到資料已經被整理的漂漂亮亮
而且是跨 `Python, Golang` 哦

```bash
ts:1695386880000000000  close:524  open:524  high:524  low:523  volume:39  code:"2330"
ts:1695386940000000000  close:523  open:524  high:524  low:523  volume:46  code:"2330"
ts:1695387000000000000  close:523  open:524  high:524  low:523  volume:36  code:"2330"
ts:1695387060000000000  close:523  open:523  high:524  low:523  volume:31  code:"2330"
ts:1695387120000000000  close:523  open:524  high:524  low:523  volume:85  code:"2330"
ts:1695387180000000000  close:523  open:523  high:524  low:523  volume:32  code:"2330"
ts:1695387240000000000  close:523  open:523  high:524  low:523  volume:27  code:"2330"
ts:1695387300000000000  close:523  open:524  high:524  low:523  volume:62  code:"2330"
ts:1695387360000000000  close:523  open:524  high:524  low:523  volume:41  code:"2330"
ts:1695387420000000000  close:524  open:523  high:524  low:523  volume:92  code:"2330"
ts:1695387480000000000  close:524  open:523  high:524  low:523  volume:47  code:"2330"
ts:1695387540000000000  close:523  open:523  high:524  low:523  volume:32  code:"2330"
ts:1695387600000000000  close:523  open:524  high:524  low:523  volume:54  code:"2330"
ts:1695387660000000000  close:524  open:524  high:524  low:523  volume:89  code:"2330"
ts:1695387720000000000  close:524  open:524  high:524  low:523  volume:43  code:"2330"
ts:1695387780000000000  close:524  open:523  high:524  low:523  volume:38  code:"2330"
ts:1695387840000000000  close:523  open:524  high:524  low:523  volume:57  code:"2330"
ts:1695387900000000000  close:524  open:524  high:524  low:523  volume:51  code:"2330"
ts:1695387960000000000  close:524  open:524  high:524  low:523  volume:102  code:"2330"
ts:1695388020000000000  close:523  open:524  high:524  low:523  volume:49  code:"2330"
ts:1695388080000000000  close:524  open:524  high:524  low:523  volume:134  code:"2330"
ts:1695388140000000000  close:524  open:524  high:524  low:523  volume:67  code:"2330"
ts:1695388200000000000  close:523  open:523  high:524  low:523  volume:74  code:"2330"
ts:1695388260000000000  close:524  open:523  high:524  low:523  volume:56  code:"2330"
ts:1695388320000000000  close:524  open:524  high:524  low:523  volume:91  code:"2330"
ts:1695388380000000000  close:523  open:524  high:524  low:523  volume:87  code:"2330"
ts:1695388440000000000  close:524  open:524  high:524  low:523  volume:51  code:"2330"
ts:1695388500000000000  close:524  open:524  high:524  low:523  volume:79  code:"2330"
ts:1695388560000000000  close:523  open:524  high:524  low:523  volume:90  code:"2330"
ts:1695388620000000000  close:524  open:524  high:524  low:523  volume:66  code:"2330"
ts:1695388680000000000  close:523  open:524  high:524  low:523  volume:127  code:"2330"
ts:1695388740000000000  close:524  open:524  high:524  low:523  volume:84  code:"2330"
ts:1695388800000000000  close:523  open:524  high:524  low:523  volume:79  code:"2330"
ts:1695388860000000000  close:524  open:524  high:524  low:523  volume:114  code:"2330"
ts:1695388920000000000  close:524  open:524  high:524  low:523  volume:90  code:"2330"
ts:1695388980000000000  close:524  open:524  high:524  low:523  volume:316  code:"2330"
ts:1695389040000000000  close:524  open:524  high:524  low:523  volume:347  code:"2330"
ts:1695389100000000000  close:524  open:524  high:524  low:523  volume:252  code:"2330"
ts:1695389400000000000  close:522  open:522  high:522  low:522  volume:6622  code:"2330"
```

## 總結

這兩天進入了基本資料、歷史資料
但都還沒進入到真正的邏輯
以及介面，開始懷疑三十天是否夠用了🤣
