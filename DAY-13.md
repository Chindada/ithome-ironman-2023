# 【鐵人賽】DAY-13 訂閱即時資料

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-13-訂閱即時資料-06.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1028/)

## 前言

昨天終於把第一個頁面簡單地呈現出來
算是為 `Flutter` 入門了一番
有了基本資料、K 線資料之後
為了完成一個最基本的交易界面
除了名字上的交易功能還差什麼？
當然就是那個一直跳動的股價
筆者就是覺得這樣真的很帥
才會步入這個世界😆

## 文件

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-13-訂閱即時資料-01.png)

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-13-訂閱即時資料-02.png)

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-13-訂閱即時資料-03.png)

文件上可以看得出來
這邊需要提供 `contract`
訂閱的資料
以永豐證券來說，可以訂閱撮合(Tick)、五檔(BidAsk)

## 再次包裝

我們先加入幾個方法
因為筆者寫文章的時候並非在股票開盤的時間
所以這邊先加入幾個期貨的 `Contract Map`，以及 `getter`

```python
def fill_future_map(self):
    for future_arr in self.__api.Contracts.Futures:
        for future in future_arr:
            with self.future_map_lock:
                self.future_map[future.code] = future
    with self.future_map_lock:
        if len(self.future_map) != 0:
            logger.info("total future: %d", len(self.future_map))
        else:
            raise RuntimeError("future_map is empty")

def get_contract_by_future_code(self, code):
    with self.future_map_lock:
        return self.future_map[code]
```

可以取得 Contract 之後
就來把訂閱也包進來

```python
def subscribe_future_tick(self, code):
    try:
        self.__api.quote.subscribe(
            self.get_contract_by_future_code(code),
            quote_type=sc.QuoteType.Tick,
            version=sc.QuoteVersion.v1,
        )
        return None
    except Exception:
        return code
```

先來跑跑看
這邊訂閱小型台指，`MXFR1`

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
    self.subscribe_future_tick("MXFR1")

    return self
```

先暫時不包裝成 `gRPC`
用之前的 Python + Golang 跑起來看看
你會發現你的終端，開始跑去一個又一個的 `Tick`
我相信沒有接觸過程式交易的人
會有一點驚喜
這是第一手的資料，如果你網路夠快
可以跟自己手上券商的 APP 來比較一下
你這邊數字會率先跳動
如下圖

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-13-訂閱即時資料-04.png)

### 問題：該如何把這些資料記錄或者往外傳？

直接在 `terminal` 把每個 `Tick` 印出來
這是券商 API 的預設行為
當初我真的還異想天開
想要去 `Parse` 每一行，來取得我要的資料
真是太沒效率了

### 文件還是要看完

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-13-訂閱即時資料-05.png)

如果繼續把剛剛的文件看完
就會發現這是可以修改的

聰明的你，應該會知道下一步就是要把這個 `Callback`
修改成一個 `gRPC`，讓這些資料可以用一個高效的方式傳到 `Golang` 那邊
因為目前的規劃，還是把所有邏輯放在 `Golang, Flutter`
最前面的這層，只是抽象化券商 API

### 先來簡單的修改一下

先簡單定義一個我們自己的 `Callback`

```python
def future_tick_callback(self, code, tick: sj.TickFOPv1):
    logger.info(
        "Time: %s, Price: %s, Volume: %s",
        datetime.strftime(tick.datetime, "%Y-%m-%d %H:%M:%S.%f"),
        tick.close,
        tick.volume,
    )
```

雖然也只是印出來
但我們先初步驗證一下這個 `Callback`
是真實有被拿去使用的
寫完方法之後
就是要照文件的教學把他 `set` 進去

```python
self.set_on_tick_fop_v1_callback(self.future_tick_callback)
```

這個時候在執行
畫面應該要比剛剛的整齊非常多
說真的，如果只是需要最新最快價格的當沖者
到這邊，再搭配券商推出的快速下單
就能夠確保你獲得的資訊比別人即時

```bash
INFO[2023-09-28 15:37:45] Time: 2023-09-28 15:37:45.020000, Price: 16357, Volume: 1
INFO[2023-09-28 15:37:45] Time: 2023-09-28 15:37:45.116000, Price: 16357, Volume: 1
INFO[2023-09-28 15:37:49] Time: 2023-09-28 15:37:48.991000, Price: 16356, Volume: 1
INFO[2023-09-28 15:37:53] Time: 2023-09-28 15:37:53.858000, Price: 16356, Volume: 6
INFO[2023-09-28 15:37:54] Time: 2023-09-28 15:37:54.308000, Price: 16356, Volume: 1
INFO[2023-09-28 15:37:57] Time: 2023-09-28 15:37:57.323000, Price: 16356, Volume: 1
INFO[2023-09-28 15:38:01] Time: 2023-09-28 15:38:01.252000, Price: 16356, Volume: 1
INFO[2023-09-28 15:38:02] Time: 2023-09-28 15:38:02.096000, Price: 16356, Volume: 1
INFO[2023-09-28 15:38:04] Time: 2023-09-28 15:38:04.631000, Price: 16356, Volume: 1
INFO[2023-09-28 15:38:04] Time: 2023-09-28 15:38:04.680000, Price: 16356, Volume: 1
```

## 總結

今天又回到 `Python` 這層
並且成功的訂閱到即時價格
很顯然並沒有完成把這些資料可以送到 `Flutter APP`
所以明天會繼續把這些資料轉化成 `gRPC`
