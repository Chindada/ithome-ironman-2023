# 【鐵人賽】DAY-15 訂閱即時資料『三』

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-14-訂閱即時資料『二』-03.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1051/)

## 前言

昨天臨時改了架構
這雖然在一般開發中
能免則免，但為了未來的擴充性
還是趁早換
畢竟 `gRPC` 雖然有提供長連線
但並不是專門拿來處理這種每秒鐘有大量數據流
今天會把 `RabbitMQ` 建立起來，並設置好

## RabbitMQ

### 啟動

這邊沒有指定任何啟動方式
筆者是習慣使用 `Docker`
官方的 [Image URL](https://hub.docker.com/_/rabbitmq)

```bash
$docker run -d \
  --restart always \
  --name rabbitmq \
  -p 5672:5672 \
  -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=password \
  rabbitmq:3.12.4-management

# result
Unable to find image 'rabbitmq:3.12.4-management' locally
3.12.4-management: Pulling from library/rabbitmq
20274425734a: Pull complete
d92e45f5d8cd: Pull complete
c27b7fd9d6be: Pull complete
88e6a5314957: Extracting [==================================================>]  10.71kB/10.71kB
bdbccf328c04: Downloading [==>                                                ]  1.058MB/20.48MB
5272b6274980: Download complete
810704aa6a10: Download complete
00c96afa4bce: Download complete
86790e5f0680: Waiting
1911143393ad: Waiting
```

提供一個範例
這邊選用的是帶有管理介面的 `management` 版本
成功啟動之後，可以訪問看看 `http://127.0.0.1:15672`
用上面啟動時給定的帳密：`admin/password`

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-13-訂閱即時資料『三』-01.png)

可以看到這個頁面就是啟動成功了

## 使用方式

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-15-訂閱即時資料『三』-02.png)

上圖是來自[官方教學](https://www.rabbitmq.com/getstarted.html)
包含了各種語言的實作，也包含到這次我們使用的 `Python, Golang`
乍看，好像其實每一種都可以符合我們需求
但依照筆者的經驗
第四種 `Routing`，在我們的情境中
是最符合的
比如說台積電的 Tick，統一就用 `2330-stock-tick` 當作 `routing`
`Python` 就往這邊送
`Golang` 就從這邊收
甚至想要多開幾個 `Golang` 來接收都不是問題

## 設置 RabbitMQ

基礎原理這邊就不多說
`RabbitMQ` 跟其他的 MQ，比如 `[Mosquitto](https://mosquitto.org)`
需要做一點前置作業
也就是建立 `Exchange`
已在台灣來說，就像是先定好要走國道一號，還是國道三號
如果 `Python, Golang` 各走一邊
那是完全不會碰到的

### RabbitMQ API

筆者習慣每次啟動後端
都會啟動一個全新的 `RabbitMQ`
這代表上次新增的 `Exchange` 需要重新建立
如果每次都手動建立
不就太不符合現在的程式部署了
所以我會透過原生 `API` 的方式去新建
在剛剛的環境可以訪問 `http://127.0.0.1:15672/api/index.html`

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-15-訂閱即時資料『三』-03.png)

上面的圖只是節錄
其中對我們來說比較重要的有兩隻

- `/api/health/checks/alarms` GET
  - 這隻可以確認 `RabbitMQ` 已經正常啟動完，可以接受後續的 `API` 操作

- `/api/exchanges` GET
  - 取得現有的 `Exchange`

- `/api/exchanges/%2F/{NAME}` PUT
  - 新增一個叫做 NAME 的 Exchange

## 實作

之前沒有講得太細節
但各位應該可以想得到
這次的專案，啟動順序是講究的
很顯然 `Python` 會是第一個
所以我將設定 `RabbitMQ` 的責任，放在 `Python` 這邊

```python
import json
import time
from base64 import b64encode

import requests

from logger import logger


class RabbitMQSetting:
    def __init__(self, rabbitmq_user: str, rabbitmq_password: str, rabbitmq_host: str, rabbitmq_exchange: str):
        self.rabbitmq_user = rabbitmq_user
        self.rabbitmq_password = rabbitmq_password
        self.rabbitmq_host = rabbitmq_host
        self.rabbitmq_exchange = rabbitmq_exchange

    def reset_rabbitmq_exchange(self):
        auth = b64encode(
            bytes(
                f"{self.rabbitmq_user}:{self.rabbitmq_password}",
                encoding="utf8",
            )
        ).decode("ascii")
        headers = {
            "Authorization": f"Basic {auth}",
            "Content-Type": "application/json",
        }

        while True:
            try:
                req = requests.get(
                    url=f"http://{self.rabbitmq_host}:15672/api/health/checks/alarms",
                    headers=headers,
                    timeout=(5, 10),
                )
            except requests.exceptions.ConnectionError:
                time.sleep(1)
                continue
            if req.status_code == 200:
                break

        req = requests.get(
            url=f"http://{self.rabbitmq_host}:15672/api/exchanges",
            headers=headers,
            timeout=(5, 10),
        )
        if req.status_code != 200:
            raise RuntimeError("RabbitMQ get exchange fail")

        exchange_arr = req.json()
        exist = False
        for ex in exchange_arr:
            if ex["name"] == self.rabbitmq_exchange:
                exist = True
                logger.warning("exchange %s already exists", self.rabbitmq_exchange)
                break

        if not exist:
            req = requests.put(
                url=f"http://{self.rabbitmq_host}:15672/api/exchanges/%2F/{self.rabbitmq_exchange}",
                data=json.dumps(
                    {
                        "type": "direct",
                        "durable": True,
                    }
                ),
                headers=headers,
                timeout=(5, 10),
            )
            logger.warning("add exchange %s", self.rabbitmq_exchange)
            if req.status_code != 201:
                raise RuntimeError("RabbitMQ exchange add fail")
```

這段可以完全的照抄🤣
放在 `main()` 的最前面

```python
if __name__ == "__main__":
    env = RequiredEnv()
    try:
        rc = RabbitMQSetting(
            env.rabbitmq_user,
            env.rabbitmq_password,
            env.rabbitmq_host,
            env.rabbitmq_exchange,
        )
        rc.reset_rabbitmq_exchange()

    except RuntimeError:
        logger.error("reset rabbitmq exchange fail, retry after 30 seconds")
        time.sleep(30)
        os._exit(0)
```

裡面的 `env` 放有我提前預訂好的 `RabbitMQ` 的相關設定
這邊可以依照自己的寫法再更動
執行起來應該會像下面提示已新增

```bash
WARN[2023-09-28 20:41:00] add exchange toc
Server started. Listening on port 50051.
```

### 檢查

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-15-訂閱即時資料『三』-04.png)

確實也成功新增了

## 總結

今天把昨天意外加入的 `RabbitMQ` 架設
也在 `Python` 上實作了增加 `Exchange`
算是搭建起串流資料的一條路
明天會把資料往者裡面送
