# 【鐵人賽】DAY-14 訂閱即時資料『二』

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-14-訂閱即時資料『二』-02.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1041/)

## 前言

昨天我們成功把即時資料取得
但只是停留在把他列印出來
今天我們要進階一點
把這邊包裝成 `gRPC` 方法
並且是一個長連線
把 `Python -> Golang` 這段資料流完成

## 定義資料結構

我們先來看一下原始的資料
這邊我會建議直接從我們的 `Terminal` 直接複製
因為官方的文件就算很頻繁更新
也許還是會有可能跟現在的資料不同

```bash
Tick(
    code='MXFJ3',
    datetime=datetime.datetime(2023, 9, 28, 17, 49, 50, 289000),
    open=Decimal('16353'),
    underlying_price=Decimal('16353.74'),
    bid_side_total_vol=6648,
    ask_side_total_vol=9824,
    avg_price=Decimal('16336.980595'),
    close=Decimal('16328'),
    high=Decimal('16362'),
    low=Decimal('16313'),
    amount=Decimal('16328'),
    total_amount=Decimal('270246333'),
    volume=1,
    total_volume=16542,
    tick_type=1,
    chg_type=4,
    price_chg=Decimal('-39'),
    pct_chg=Decimal('-0.238284'),
    simtrade=0
    )
```

從上面的這段
很清楚地看出型別
也可以回到官方文件在做一次 `Double check`
[先來定義 `Protobuf` 如下](#future-protobuf)

```c
message FutureRealTimeTickMessage {
    string code              = 1;
    string date_time         = 2;
    double open              = 3;
    double underlying_price  = 4;
    int64 bid_side_total_vol = 5;
    int64 ask_side_total_vol = 6;
    double avg_price         = 7;
    double close             = 8;
    double high              = 9;
    double low               = 10;
    double amount            = 11;
    double total_amount      = 12;
    int64 volume             = 13;
    int64 total_volume       = 14;
    int64 tick_type          = 15;
    int64 chg_type           = 16;
    double price_chg         = 17;
    double pct_chg           = 18;
    bool simtrade            = 19;
}
```

## 定義 gRPC 方法

前面定義完格式
現在該來定義方法

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-14-訂閱即時資料『二』-01.gif)

來思考一下
目前我們執行訂閱是在 `Python` 成功登入後
就自行訂閱起來
但，這很明顯不符合未來我們想要從 APP 端發動訂閱
所以第一步先建立一個 gRPC 通道
並且是由 `Golang` 這邊發動
建立後，由 `Python` 這邊不斷地發送 `Tick`

### 但是

這邊我就幫各位走了一小條彎路
本來想得很美好
把訂閱來的資料收在一個 `Queue` 裡面
然後由 `Golang` 這邊建一個通道
`Python` 這邊就從這個 `Queue` 裡面拿資料，沒資料就 `block`
達成有資料就會第一時間送
這在只訂閱一檔標的的時候可行
訂閱多檔，就會發現資料量太大，被阻塞了
這次鐵人賽雖然是看起來只訂閱一檔
但未來可能不一定嘛

## 導入新元件

訂閱多檔這件事
可以很容易跟 `MQTT` 這個協議聯想在一起
我來請出 `ChatGPT` 來解釋一下

> MQTT（Message Queuing Telemetry Transport）是一種輕量級的通訊協定，特別適用於低帶寬、不穩定網路環境下的物聯網（IoT）應用。以下是 MQTT 的幾個優點：
>
> 1. **輕量級和高效能**：
>     MQTT 是一種輕量級協定，傳輸的數據包裝較小，並且協定本身的開銷也很小。這使得它在資源有限的設備和網路上運作高效，並減少了能耗和網路流量。
>
> 3. **實時通訊**：
>     MQTT 允許即時的雙向通訊，使得系統能夠快速地回應事件、狀態變化或指令。這對於需要及時互動的 IoT 應用非常重要。
>
> 5. **網路斷線恢復**：
>     MQTT 具有網路斷線恢復機制，當設備在網路斷線後重新連接時，將恢復連線並接收遺漏的訊息。這保證了訊息的可靠傳遞。
>
> 7. **異步通訊**：
>     MQTT 支援異步通訊，設備之間可以非同步地交換資訊。這種特性允許設備以自身的節奏進行通訊，減少了資源浪費。
>
> 9. **訂閱/發佈模型**：
>     MQTT 採用訂閱/發佈模型，可以輕鬆地向多個設備發送資訊，並且這些設備可以針對特定主題進行訂閱，只接收與其相關的資訊，提高了系統的靈活性和效率。
>
> 11. **安全性**：
>     MQTT 支援使用 TLS/SSL 加密來保護數據傳輸，這使得資料在傳遞過程中得以保密和安全。
>
> 13. **廣泛的應用**：
>     MQTT 已經成為 IoT 領域的標準協定，並且被廣泛運用於智慧家居、工業自動化、健康醫療等各種領域。
>
> 總的來說，MQTT 以其輕量、高效、實時、彈性和安全等特性成為 IoT 系統中一個非常有用的通訊協定。
>
> Using ChatGPT

利用『**訂閱/發佈模型**』
可以做到訂閱多檔標的
那我們選的 MQTT Server 會是哪一款呢～～

## RabbitMQ

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-14-訂閱即時資料『二』-02.png)

太細節的 `Code` 我就不會發上來
主要是給出來一個概念

加入這個之後，我們的架構圖就會變成下面這樣

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-14-訂閱即時資料『二』-03.png)

## 總結

真的是峰迴路轉
本來今天以為可以很順利寫到串完資料
但我也不想給一個效果很差的東西
所以臨時又改了架構
明天再來好好的寫寫我這邊的實作
