# 【鐵人賽】DAY-12 Flutter 第一個介面『三』

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-12-Flutter-第一個介面『三』-05.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/954/)

## 前言

好事多磨
今天總算要進入畫面呈現了
前天有利用 `Flutter` 的預設指令創建了一個預設專案
並且把它跑在 `iOS` 模擬器上
額外加了第一個按鈕
並且呼叫我們自己開的 `API`
昨天覺得有必要補充一些基本知識
今天來把它畫出來吧

## 呈現的方法

![【鐵人賽】DAY-10-Flutter-第一個介面](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-10-Flutter-第一個介面-04.png)

上面是資料的樣貌
是一千多筆有多個欄位的資料型態
在手機小小的螢幕上
勢必不可能像電腦螢幕
用一個大大的表格來呈現
**格狀**、**列表**、**標籤分頁**都是可行的方式
這邊我就先選用列表來呈現
自己刻就是想怎樣就怎樣～
但目前的 APP 佈局還是預設的那樣（下圖）
需要做一些調整

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-12-Flutter-第一個介面『三』-01.png)

## 加入 AppBar

這邊先不考慮前後頁、路由等等
先當作這個 APP 只有這一頁

```dart
class _MainAppState extends State<MainApp> {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Stock List'),
        ),
        body: const Center(
          child: Text('Hello World'),
        ),
      ),
    );
  }
}
```

很簡單的一個 `AppBar`
只有一個 `title`
但還記得我們之前有一個[方法](https://tocandraw.com/2023-ironman/916/#fetch-method)
是可以去呼叫 API 的
讓我們把它放在右上角

```dart
class _MainAppState extends State<MainApp> {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Stock List'),
          actions: const [
            IconButton(
              icon: Icon(Icons.refresh),
              onPressed: fetchStockDetail,
            ),
          ],
        ),
        body: const Center(
          child: Text('Hello World'),
        ),
      ),
    );
  }
}
```

畫面就會變成下面這樣

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-12-Flutter-第一個介面『三』-02.png)

看起來就有點像一回事了
可以試著點擊看看上面的 `Refresh` 按鈕
你的 `console` 應該會像第一天一樣
印出股票基本資料

## FutureBuilder

再繼續之前
先來介紹一下 `FutureBuilder`
昨天有提到 `Flutter` 當中 Widget 有分兩種
分別為 `StatefulWidget` 以及 `StatelessWidget`
現在這種畫面設計
很顯然是有狀態的
下面來加入 `FutureBuilder`

```dart
class _MainAppState extends State<MainApp> {
  Future<List<StockDetail>> stockArray = Future<List<StockDetail>>.value(<StockDetail>[]);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Stock List'),
          actions: const [
            IconButton(
              icon: Icon(Icons.refresh),
              onPressed: fetchStockDetail,
            ),
          ],
        ),
        body: FutureBuilder<List<StockDetail>>(
          future: stockArray,
          builder: (context, snapshot) {
            return const Center(
              child: CircularProgressIndicator(
                color: Colors.black,
              ),
            );
          },
        ),
      ),
    );
  }
}
```

幾點說明一下

```dart
Future<List<StockDetail>> stockArray = Future<List<StockDetail>>.value(<StockDetail>[]);
```

宣告了一組空的 `Future`
並且在 `Scaffold` 的 body 中放入 `FutureBuilder`
而 `stockArray` 則作為他的顯示資料
只要資料有任何變化，會馬上變更畫面
現在刻意讓他顯示等待的轉圈圈

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-12-Flutter-第一個介面『三』-03.png)

## ListView

前面有提到預計使用列表來呈現
很顯然我們的資料會超過手機長度
最適合的就是 `ListView`
細節怎麼用，可以去翻翻看文件
先看整合起來的 `Code`

```dart
class _MainAppState extends State<MainApp> {
  Future<List<StockDetail>> stockArray = Future<List<StockDetail>>.value(<StockDetail>[]);

  fetchStockDetail() async {
    final stockArr = <StockDetail>[];
    try {
      final response = await http.get(Uri.parse('http://127.0.0.1:8080/basic-data/stock-data'));
      if (response.statusCode == 200) {
        var stockData = jsonDecode(response.body) as Map<String, dynamic>;
        for (final i in stockData['data']['stock']) {
          stockArr.add(StockDetail.fromJson(i as Map<String, dynamic>));
        }
        setState(() {
          stockArray = Future<List<StockDetail>>.value(stockArr);
        });
      }
    } on Exception {
      throw Exception('Failed to load stock data');
    }
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: Scaffold(
        appBar: AppBar(
          title: const Text('Stock List'),
          actions: [
            IconButton(
              icon: const Icon(Icons.refresh),
              onPressed: fetchStockDetail,
            ),
          ],
        ),
        body: FutureBuilder<List<StockDetail>>(
          future: stockArray,
          builder: (context, snapshot) {
            if (snapshot.hasData && snapshot.data!.isNotEmpty) {
              return ListView(
                children: snapshot.data!.map((stock) {
                  return Card(
                    child: ListTile(
                      title: Text(stock.name!),
                      subtitle: Text(stock.code!),
                      trailing: Text(stock.reference.toString()),
                    ),
                  );
                }).toList(),
              );
            }
            return const Center(
              child: CircularProgressIndicator(
                color: Colors.black,
              ),
            );
          },
        ),
      ),
    );
  }
}
```

做了一點改變
把 `fetchStockDetail` 移入 `class` 當中
變成一個類方法
可以直接去影響剛剛宣告的 `array`
試試按下右上角的 `Refresh`
是不是長得下面這樣呢

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-12-Flutter-第一個介面『三』-04.png)

## 總結

總算是刻出第一個畫面
而且前端、後端都是自己來👏
打通第一條資料流
接下來幾天要進入呈現一些歷史資料
