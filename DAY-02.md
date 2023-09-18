# 【鐵人賽】DAY-02 從架構圖開始

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-01.png)

本文同步發佈於[毛毛的踩坑人生](https://blog.tocandraw.com/2023/09/17/2023-ironman/767/timhsu/)

## 前言

昨天大致定下目標，那今天就要來大致想一下整個服務的藍圖、架構圖
架構圖我一律推薦 [C4 model](https://c4model.com)

> ### Maps of your code
>
> The C4 model was created as a way to help software development teams describe and communicate software architecture, both during up-front design sessions and when retrospectively documenting an existing codebase. It's a way to create maps of your code, at various levels of detail, in the same way you would use something like Google Maps to zoom in and out of an area you are interested in.
>
> 原文是 Maps of your code，這段話，我覺得很傳神
> 沒有方向，是很容易做不出來想要的程式

## 開始規劃

在開始規劃之前，我先推薦各位一個工具

就是各位熟悉的 draw.io，推薦的原因其中也包括
他現在已經內建 C4 Model，以前都還額外裝呢

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-02-1024x860.png)

1. 首先，我們會從券商的 API 開始串接，在 C4 當中，灰色是代表外部的系統

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-03.png)

2. 再來我們要加入我們的第一個組件，這邊預計他會是一隻由 Python 攥寫的程式
    其主要作用會是包裝券商的 API，並且轉化成 `gRPC Method`
    在包裝的過程當中，也可以些許的加入一些自己的邏輯
    現在暫時稱它叫做 `Python gRPC Forwarder`
    取這個名稱的原因無他，因為他就只是做一個轉發並轉換成 gRPC

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-04.png)

再來稍微展開一下
這隻程式需要做到什麼

- 下單

- 取得即時資料

- 取得歷史資料

- 將以上 API 轉化成 gRPC

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-07-1024x461.png)

這樣做還有一個好處，我相信未來各個券商開放自己 API 是一個趨勢
如果橫向擴展，儘管券商 API 格式都不相同
但我們可以透過事先定義好的 `gRPC, protobuf`，就是常聽到的 `統一 Interface`

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-05.png)

3. 前面我們開好了 `gRPC Server`，那現在就要加入主要使用 Golang 攥寫的後端服務

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-06.png)

一樣，我們展開一下

- 需要 RESTful API

- 可能也需要一個長連線，所以就開一個 WEB Socket Server

- 需要串接剛剛開好的 gRPC Server

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-08-1024x859.png)

4. 最後，當然就是加入使用者介面跟使用者

![IMG](https://blog.tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-02-從架構圖開始-09-344x1024.png)

## 總結

今天大致規劃出需要實作的部分，會有三個專案
包括兩個後端、一個前端
其實有跳過一些東西，比如為什麼要選用 `Python, Golang and Flutter`
我會在未來的篇幅中，好好的來講一講
