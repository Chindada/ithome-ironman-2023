# 【鐵人賽】DAY-07 從券商 API 開始『二』

![IMG](https://tocandraw.com/wp-content/uploads/2022/04/Golang_Python_Trade_Cover_2.png.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/851/)

## 前言

昨天把需要做的功能
雖然不算很細緻
但已經知道一些方向
那今天就要來進入真正的開篇
來挖挖看
在券商的 API 裡面，有多少寶藏可以挖😆

## 首先！基本資料

這邊我就不賣關子了
一定有很多人心裡想
平常買股票、期貨，哪會在意這標的物的基本資料
當然還是有很多人會去看基本面
這邊指的是 API 裡面去怎麼認識他的目標
以 `Shioaji` 來說就是 [Contract](https://sinotrade.github.io/tutor/contract/)
玉山證券的 `Fugle`，雖然文件上寫的是直接用股票代號
但像是 KY、ETF 等等，這些不能很確定要傳什麼的
還是透過 API 提供的資料最準確

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-07-從券商-API-開始『二』-01.png)

上面的圖是官方的範例
同樣，我們來包裝一下

```python
def __init__(self):
    self.__api = sj.Shioaji()
    self.__login_status_lock = threading.Lock()
    self.__login_progess = int()

    self.stock_map: dict[str, Contract] = {}
    self.stock_map_lock = threading.Lock()
```

在之前範例中的 `Shioaji Class`
新增一個 map，拿來存放所有、或者我們有興趣的股票
看 Code

```python
def fill_stock_map(self):
    for contract_arr in self.__api.Contracts.Stocks:
        for contract in contract_arr:
            if contract.day_trade == sc.DayTrade.Yes.value and contract.category != "00":
                with self.stock_map_lock:
                    self.stock_map[contract.code] = contract
                    logger.info("add stock: %s %s", contract.code, contract.name)
    with self.stock_map_lock:
        if len(self.stock_map) != 0:
            logger.info("total stock: %d", len(self.stock_map))
        else:
            raise RuntimeError("stock_map is empty")
```

小小說明一下
我做了一點篩選
把 category 是 00 的過濾掉，比如加權指數就在這類
這不能交易，也就沒有存起來的必要
另外，我也過濾掉不能夠當沖的股票
這邊並不是鼓勵當沖，而是程式交易，剛開始難免出錯
至少要能夠沖掉、回補才不會造成大傷害

來跑跑看，累積一點成就感
不然到第七天，交易的影子都沒有🤣

```bash
...
...
INFO[2023-09-22 21:48:02] add stock: 8210 勤誠
INFO[2023-09-22 21:48:02] add stock: 2509 全坤建
INFO[2023-09-22 21:48:02] add stock: 6192 巨路
INFO[2023-09-22 21:48:02] add stock: 3005 神基
INFO[2023-09-22 21:48:02] add stock: 4104 佳醫
INFO[2023-09-22 21:48:02] add stock: 2484 希華
INFO[2023-09-22 21:48:02] add stock: 1225 福懋油
INFO[2023-09-22 21:48:02] add stock: 8070 長華*
INFO[2023-09-22 21:48:02] add stock: 2889 國票金
INFO[2023-09-22 21:48:02] add stock: 1437 勤益控
INFO[2023-09-22 21:48:02] add stock: 6269 台郡
INFO[2023-09-22 21:48:02] add stock: 3673 TPK-KY
INFO[2023-09-22 21:48:02] add stock: 9931 欣高
INFO[2023-09-22 21:48:02] add stock: 6139 亞翔
INFO[2023-09-22 21:48:02] add stock: 1231 聯華食
INFO[2023-09-22 21:48:02] add stock: 2364 倫飛
INFO[2023-09-22 21:48:02] add stock: 2466 冠西電
INFO[2023-09-22 21:48:02] add stock: 1583 程泰
INFO[2023-09-22 21:48:02] add stock: 3266 昇陽
INFO[2023-09-22 21:48:02] add stock: 9944 新麗
INFO[2023-09-22 21:48:02] add stock: 2845 遠東銀
INFO[2023-09-22 21:48:02] add stock: 2431 聯昌
INFO[2023-09-22 21:48:02] add stock: 1609 大亞
INFO[2023-09-22 21:48:02] total stock: 1524
```

有足足 1524 之多～

## 查詢基本資料

```python
def get_contract_by_stock_num(self, num):
    if num == "tse_001":
        return self.__api.Contracts.Indexs.TSE.TSE001

    if num == "otc_101":
        return self.__api.Contracts.Indexs.OTC.OTC101

    with self.stock_map_lock:
        return self.stock_map[num]
```

這邊再多一個方法，可以查詢
同時也偷渡了一個小地方
雖然剛剛說加權指數不能交易，所以不存起來
但昨天其實是很明確的說要查詢大盤
這邊還是回去 API 裡面拿

至於 map 的讀寫鎖 `stock_map_lock`
我認為在這種雜湊的結構
用鎖，是一個良好的習慣
完成這個方法之後，之後就可以有一些應用了～

## 總結

今天初次踏入了一些股票的領域
如果是第一次接觸的人
我相信平常寫習慣了公司專案
突然看到自己寫的程式，原來可以跟生活這麼近
應該是充滿雀躍的吧
