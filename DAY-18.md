# 【鐵人賽】DAY-18 使用 Golang 建立 WebSocket

![IMG](https://tocandraw.com/wp-content/uploads/2022/04/Golang_Python_Trade_Cover_2.png.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1083/)

## 前言

這篇文章，是在日本寫的哦
前幾天都是預先寫好的庫存😝
回到正題，前幾天順利地把即時價格送到了 `Golang` 這邊
現在最後一步，就是把這個資料再送到 `APP`
如果你完全不需要介面
那送到 `Golang` 就可以進行後續的自動下單
但這次我想帶著各位一起把下單介面做出來

## WebSocket of Gin

如同標題寫的
會延續我們開出來的第一支 [`API`](https://tocandraw.com/2023-ironman/895/#first-golang-restful-api)
當時使用的第三方套件是 [`gin`](https://github.com/gin-gonic/gin)
通常 `RESTful API` 跟 `WebSocket` 的入口會是一樣的
也是有人不是這樣
但我傾向把 `WebSocket` 加入 `gin` 所使用的 `Web Server`
下面就開始吧
也可以參考我在 [2022.3.1 寫的文章](https://tocandraw.com/coding/golang/111/)
可能略有不同，但大致上是同樣脈絡的

### 包裝 [Gorilla WebSocket](https://github.com/gorilla/websocket)

```bash
go get github.com/gorilla/websocket
```

如題，這次選用的 `package` 還是 `gorilla`
先安裝起來吧
記得在專案資料夾跑以下指令

```go
type WSRouter struct {
 msgChan chan interface{}
 conn    *websocket.Conn
 ctx     context.Context
 logger  Logger
}
```

說明一下

- `msgChan`：一個拿來存放這個 `WebSocket` 所有的訊息的 `Channel`

- `conn`：使用者連上 `Gin` `API` 之後被升級的連線

- `ctx`：呈上，使用者使用的連線所使用的 `Context`，可以透過這個 `Context` 來判斷 `Client` 是否還在線

- `logger`：就是 `logger`

```go
func NewWSRouter(c *gin.Context, logger Logger) *WSRouter {
 r := &WSRouter{
  msgChan: make(chan interface{}),
  ctx:     c.Request.Context(),
  logger:  logger,
 }
 r.upgrade(c)
 return r
}

func (w *WSRouter) upgrade(gin *gin.Context) {
 upGrader := websocket.Upgrader{
  CheckOrigin: func(r *http.Request) bool {
   return true
  },
  ReadBufferSize:  1024,
  WriteBufferSize: 1024,
 }

 c, err := upGrader.Upgrade(gin.Writer, gin.Request, nil)
 if err != nil {
  w.logger.Errorf("upgrade websocket err: %v", err)
  return
 }

 w.conn = c
 go w.write()
}


func (w *WSRouter) write() {
 for {
  select {
  case <-w.Ctx().Done():
   return

  case cl := <-w.msgChan:
   switch v := cl.(type) {
   case string:
    w.sendText([]byte(v))

   case *pb.WSMessage:
    if serveMsgStr, err := proto.Marshal(v); err == nil {
     w.sendBinary(serveMsgStr)
    }

   default:
    if serveMsgStr, err := json.Marshal(v); err == nil {
     w.sendText(serveMsgStr)
    }
   }
  }
 }
}

func (w *WSRouter) sendText(data []byte) {
 _ = w.conn.WriteMessage(websocket.TextMessage, data)
}

func (w *WSRouter) sendBinary(data []byte) {
 _ = w.conn.WriteMessage(websocket.BinaryMessage, data)
}

func (w *WSRouter) ReadFromClient(forwardChan chan []byte) {
 for {
  _, message, err := w.conn.ReadMessage()
  if err != nil {
   close(forwardChan)
   break
  }

  if string(message) == "ping" {
   w.msgChan <- "pong"
   continue
  }

  forwardChan <- message
 }

 if err := w.conn.Close(); err != nil {
  w.logger.Errorf("websocket close err: %v", err)
 }
}

func (w *WSRouter) SendToClient(msg interface{}) {
 w.msgChan <- msg
}

func (w *WSRouter) Ctx() context.Context {
 return w.ctx
}
```

上面這一部分就是我包裝的方式，就可以把 `gin` 的 `Context` 傳進來
建立完成之後 `(New)`，記得還要提供一個訊息交換的 `Channel` 給 `ReadFromClient`
可以看得出來，無論是 `write, read` 都是用利用 `Golang` 原生的 `Channel`
來做一個阻塞，保證讀寫的順序以及互斥
下面還是簡單來一個範例

### 簡單示範

前面有了將 `gin` 連線升級的方法
很明顯，就是要在使用者呼叫原本 `gin` 上的路由時
讓這個連線進到另一個空間

#### 來一個新的路由

```go
handler := gin.New()
handler.Use(gin.Recovery())
wsRoute := handler.Group("/ws")
wsRoute.GET("/time", func(c *gin.Context) {
 httpserver.NewWSRouter(c, logger)
})
```

這邊開了一個新的 `gin router`
在 `/ws/time`
但目前還沒有塞入任何邏輯
僅是將連線升級

#### 加入資料

```go
wsRoute.GET("/time", func(c *gin.Context) {
 w := httpserver.NewWSRouter(c, logger)
 forwardChan := make(chan []byte)
 go func() {
  for {
   select {
   case msg := <-forwardChan:
    if string(msg) == "time" {
     w.SendToClient(fmt.Sprintf("Current time: %v", time.Now()))
    }

   case <-w.Ctx().Done():
    return
   }
  }
 }()
 w.ReadFromClient(forwardChan)
})
```

#### 驗證

這邊一樣使用相同的[工具](https://chrome.google.com/webstore/detail/piesocket-websocket-teste/oilioclnckkoijghdniegedkbocfpnip)
先連連看

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-18-使用-Golang-建立-WebSocket-01.png)

然後試試看 `send "time"`
按照剛剛的小邏輯
應該要回復現在時間

![IMG](https://tocandraw.com/wp-content/uploads/2023/10/【鐵人賽】DAY-18-使用-Golang-建立-WebSocket-02.png)

看起來是成功了
送了兩次 `time`
分別回傳不同時間

## 總結

雖然這篇有點舊文重發的感覺
但其實內容還還是有一點點變化
今天把通道建立起來
接下來只要把資料放進去
讓 `Flutter` 那邊可以接
就可以在 `APP` 看到價格的跳動了
