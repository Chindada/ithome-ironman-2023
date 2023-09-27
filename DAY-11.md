# 【鐵人賽】DAY-11 Flutter 第一個介面『二』

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-11-Flutter-第一個介面『二』-01.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/934/)

## 前言

昨天停留再利用 `Flutter` 去呼叫我們在前幾天串好的
`Python(gRPC Server) -> Golang(gRPC Client、http Server) -> Flutter`
今天就會試著把這些資料顯示在 APP 頁面上
如果你是第一次開發 APP
我相信這比完成線上那些固定的範例還要激動人心
雖然這部分都還未跟交易扯上關係
但我認為拿來暖身是一個不錯的選擇

## Widget 種類

眼尖的人一定會發現
在昨天的示範裡面的 [MainApp](https://tocandraw.com/2023-ironman/916/#button-example)
怎麼跟 `flutter create` 的不太一樣
因為我有偷偷換成今天所需要的東西
這邊就要先來講兩個東西
在 `Flutter` 當中所有的組件都被稱作『Widget』
這又被分成兩大類

### StatelessWidget 無狀態

```dart
class StatelessExample extends StatelessWidget {
  const StatelessExample({super.key});

  @override
  Widget build(BuildContext context) {
    return const Placeholder();
  }
}
```

顧名思義，他沒有狀態
無論 APP 顯示的資料有任何變化，畫面在一開始渲染完
就固定住了，不會再有任何變化
我講的很抽象
可以參考官方的說明

<https://youtu.be/wE7khGHVkYY>

### StatefulWidget 有狀態

```dart
class StatefulExample extends StatefulWidget {
  const StatefulExample({super.key});

  @override
  State<StatefulExample> createState() => _StatefulExampleState();
}

class _StatefulExampleState extends State<StatefulExample> {
  @override
  Widget build(BuildContext context) {
    return const Placeholder();
  }
}
```

相反的，在 `StatefulWidget` 當中
我們是可以給這個 `Widget` 一點『刺激』🤣
無論是點擊、下拉、API 的資料變化
都可以讓已經渲染好的畫面
在重新渲染一次
當然最重要的是，他可以把狀態存起來
跟後續的刺激再繼續互動
一樣可以參考一下官方說明

<https://youtu.be/AqCMFXEmf3w>

## Widget 種類

就跟 `Web` 元件一樣
在 `Flutter` 當中，`Widget` 也是包羅萬象
筆者所知道的也只是冰山一角
這邊就不獻醜，想要看看有哪些可以到[官方](https://docs.flutter.dev/ui/widgets)看看
裡面有一些也會有線上的練習

## 回到正題

昨天已經把資料用 `Flutter` 的 `http` 套件呼叫我們自己開的 API
把股票的基本資料要回來

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-10-Flutter-第一個介面-04.png)

我們要顯示出來這些資料
並沒有強制一定要用 StatefulWidget 才能顯示
也可以將所有資料準備就緒
再用一個 StatelessWidget 把資料顯示出來
沒有規定要怎麼做
具體還是要看場景
但，在 `Flutter` 當中，StatelessWidget 是相對比較省效能的
因為他不需要記住狀態，只要被渲染過一次，後續如果使用者呼叫
也無需重新渲染，還可以拿出來重複使用

### Scaffold

各位應該有注意到
自動生成的 Sample Code 當中
扣除掉最上層的 MaterialApp（這是 Google 的 Material Design 樣式）
接下來就會看到 `Scaffold`
這是最基本的 APP 『骨架』
它包含了

1. **AppBar**

3. **Body**

5. **BottomNavigationBar**

7. **Drawer**

我僅是寫出最常見的四大部分
他們佔據了畫面的上、中、下、側邊欄
在後續的天數當中
會不斷地看到這個框架

## 總結

本來要今天直接把畫面顯示出來
但昨天不小心漏了說明我把 StatelessWidget 換成 `StatefulWidget`
總覺得還是得花一點時間大致說明一下
明天再接續著把第一個介面完成
然後就可以一個一個功能把它完成
