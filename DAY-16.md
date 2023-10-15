# 【鐵人賽】DAY-16 訂閱即時資料『四』

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-15-訂閱即時資料『三』-04.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1060/)

## 前言

昨天完成了新加入的 `RabbitMQ` 的啟動設置
真的真的要把訂閱資料往上層的 `Golang` 送了
完成這部分之後
在後端的技術債就算是還完了
剩下的交易功能，其實跟前幾天的基本資料傳遞
是一模壹樣的

## gRPC 方法

前天我們定義好了 `Protobuf`
再來展示一次

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

接著輪到要定義 `rpc` 的時候
才發現問題，臨時加入了 `RabbitMQ`

### `RabbitMQ on Python`

我們首先要新增一個 `class`
用來連線 `RabbitMQ`
也會在裡面定義好券商 `API` 所需要的 `Callback`
這樣每當訂閱資料一到，就會自動使用這個 `RabbitMQ` 連線
把資料送出去

```python
import threading
import pika

from queue import Queue

from pika.channel import Channel

class PikaCC:
    def __init__(self, conn: pika.BlockingConnection, channel: Channel):
        self.conn = conn
        self.channel = channel

    def heartbeat(self):
        self.conn.process_data_events()


class RabbitMQS:
    def __init__(self, url: str, exchange: str, pool_size: int):
        self.parameters = pika.URLParameters(url)
        self.exchange = exchange
        self.pool_size = pool_size

        # rabbit mq connection queue
        self.pika_queue: Queue = Queue()

        # initial connections
        self.fill_pika_queue()

        # lock
        self.order_cb_lock = threading.Lock()

    def create_pika(self):
        conn = pika.BlockingConnection(self.parameters)
        channel = conn.channel()
        channel.exchange_declare(exchange=self.exchange, exchange_type="direct", durable=True)
        return PikaCC(conn, channel)

    def fill_pika_queue(self):
        for _ in range(self.pool_size):
            self.pika_queue.put(self.create_pika())
```

說明一下
`Pika` 是 `Python` 這邊連 `RabbitMQ` 的一個套件
他需要的參數有三個

- `RabbitMQ URL`: 會長這樣子- > `amqp://admin:password@127.0.0.1:5672/%2f?heartbeat=0`
  - heartbeat=0 是相當重要的參數

  - 在我們的場景下，確保最高效能，使用一個以上的連線

  - 如果沒有設置這個，會因為閒置，而被強制斷線

- `exchange`： 就是前面提到的預先定義好的道路總稱

- `pool_size`：剛剛有提到的，我們一共會建立多少連線

### Callback

```python
def future_quote_callback_v1(self, _, tick: sj.TickFOPv1):
    rabbit = self.pika_queue.get(block=True)
    try:
        rabbit.channel.basic_publish(
            exchange=self.exchange,
            routing_key=f"future_tick:{tick.code}",
            body=mq_pb2.FutureRealTimeTickMessage(
                code=tick.code,
                date_time=datetime.strftime(tick.datetime, "%Y-%m-%d %H:%M:%S.%f"),
                open=tick.open,
                underlying_price=tick.underlying_price,
                bid_side_total_vol=tick.bid_side_total_vol,
                ask_side_total_vol=tick.ask_side_total_vol,
                avg_price=tick.avg_price,
                close=tick.close,
                high=tick.high,
                low=tick.low,
                amount=tick.amount,
                total_amount=tick.total_amount,
                volume=tick.volume,
                total_volume=tick.total_volume,
                tick_type=tick.tick_type,
                chg_type=tick.chg_type,
                price_chg=tick.price_chg,
                pct_chg=tick.pct_chg,
                simtrade=tick.simtrade,
            ).SerializeToString(),
        )
    except Exception as err:
        logger.error("future_quote_callback_v1 error %s", err)
    self.pika_queue.put(rabbit)
```

說明一下
昨天有提到，我們所選的模式是 `Routing Mode`
需要一個 `routing_key`，這邊就是 `future_tick:CODE`
每一個股票、期貨會有自己的 `key`
互相不衝突

### 運作流程

每當有資料從券商進來時
`rabbit = self.pika_queue.get(block=True)`
會先去取得一個連線
如果全部都被佔用，則會 `block` 住

傳出去的資料格式就依照上面定義好的 `mq_pb2.FutureRealTimeTickMessage`
搭配上 `SerializeToString()`

## 整合

先醜醜的包裝一下

```python
class Shioaji:
    def __init__(self):
        self.__api = sj.Shioaji()
        self.__login_status_lock = threading.Lock()
        self.__login_progess = int()

        self.stock_map: dict[str, Contract] = {}
        self.stock_map_lock = threading.Lock()
        self.future_map: dict[str, Contract] = {}
        self.future_map_lock = threading.Lock()

        env = RequiredEnv()
        rabbit = RabbitMQS(
            env.rabbitmq_url,
            env.rabbitmq_exchange,
            64,
        )
        self.rabbit = rabbit
```

在初始化的時候，去連了 `RabbitMQ`
並且在登入過後
設定 `Callback`

`self.set_on_tick_fop_v1_callback(self.rabbit.future_quote_callback_v1)`

### 結果

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-15-訂閱即時資料『三』-05.png)

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-15-訂閱即時資料『三』-06.png)

## 總結

從上圖來看
每次券商發即時資料時
確實都有透過 `RabbitMQ` 來送資料
這樣就完成一半了
明天就可以來準備從 `Golang` 那邊收～
