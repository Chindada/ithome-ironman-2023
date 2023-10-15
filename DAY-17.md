# 【鐵人賽】DAY-17 訂閱即時資料『五』

![IMG](https://tocandraw.com/wp-content/uploads/2022/04/Golang_Python_Trade_Cover_2.png.png)

本文同步發佈於[毛毛的踩坑人生](https://tocandraw.com/2023-ironman/1068/)

## 前言

昨天很成功的把券商來的即時資料
送進新加入的 `RabbitMQ`
這條資料流已經完成三分之一
剩下的另外三分之二的其中之一
就是從 `Golang` 接收
有資料就可以做很多事情
我們可以算一些券商沒有給我們的資料
再開接口給 `Flutter` 去接

## Golang 連接 RabbitMQ

我們在 Golang 使用的 libary 是這套 [amqp091-go](https://github.com/rabbitmq/amqp091-go#go-rabbitmq-client-library)
一樣，把它包裝一下

```go
package rabbitmq

import (
 "fmt"
 "time"

 amqp "github.com/rabbitmq/amqp091-go"
)

const (
 _defaultConnAttempts = 10
 _defaultConnWaitTime = 5
)

type Logger interface {
 Infof(format string, args ...interface{})
 Errorf(format string, args ...interface{})
}

type Connection struct {
 exchange string
 url      string
 waitTime time.Duration
 attempts int

 conn   *amqp.Connection
 logger Logger
}

func New(exchange, url string, opts ...Option) (*Connection, error) {
 conn := &Connection{
  exchange: exchange,
  url:      url,
  waitTime: time.Duration(_defaultConnWaitTime) * time.Second,
  attempts: _defaultConnAttempts,
 }

 for _, opt := range opts {
  opt(conn)
 }

 if err := conn.connect(); err != nil {
  return nil, err
 }

 return conn, nil
}

func (c *Connection) connect() error {
 var err error
 for c.attempts > 0 {
  if c.conn, err = amqp.Dial(c.url); err == nil {
   break
  }

  c.attempts--
  if err != nil && c.attempts == 0 {
   return err
  }
  c.Infof("RabbitMQ is trying to connect, attempts left: %d\n", c.attempts)
  time.Sleep(c.waitTime)
 }
 return nil
}
```

這是我習慣的包裝方式
就不太細節的解釋
這一段 `New` 一個 `Connection` 出來以後
其實也就跟昨天 `Python` 一樣
只是連上 `RabbitMQ`
還有一些事情需要做
才能夠正常獲取資料

### 接受方法

還記得昨天定義好的 `routing_key` 嗎？
這邊就派上用場了
下面這段會綁定 `Exchange` 以及 `routing_key`

```go
func (c *Connection) BindAndConsume(key string) (<-chan amqp.Delivery, error) {
 channel, err := c.conn.Channel()
 if err != nil {
  return nil, err
 }
 err = channel.ExchangeDeclare(
  c.exchange,
  "direct",
  true,
  false,
  false,
  false,
  nil,
 )
 if err != nil {
  return nil, err
 }
 queue, err := channel.QueueDeclare(
  "",
  false,
  false,
  true,
  false,
  nil,
 )
 if err != nil {
  return nil, err
 }
 err = channel.QueueBind(
  queue.Name,
  key,
  c.exchange,
  false,
  nil,
 )
 if err != nil {
  return nil, err
 }
 delivery, err := channel.Consume(
  queue.Name,
  "",
  true,
  false,
  false,
  false,
  nil,
 )
 if err != nil {
  return nil, err
 }
 return delivery, nil
}
```

## 整合

下面這段演示了
建立連線 -> 綁定某一個 `routing_key`
在這個 `Case` 就是 `future_tick:MXFJ3`
成功綁定後
在一個 `for loop`
把資料印出來

```go
rabbit, err := rabbitmq.New(
 "toc",
 "amqp://admin:password@127.0.0.1:5672/%2f",
)
if err != nil {
 log.Fatalf("could not connect to rabbitmq: %v", err)
}

delivery, bErr := rabbit.BindAndConsume("future_tick:MXFJ3")
if bErr != nil {
 log.Fatalf("could not bind and consume: %v", bErr)
}
for {
 d, opened := <-delivery
 if !opened {
  return
 }
 fmt.Println(d.Body)
}
```

如果成功跑起來
是不是發現哪裡怪怪的.....
為什麼看不懂

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-17-訂閱即時資料『五』-01.png)

應該已經有人發現原因了
因為從 `Python` 那側發送出來的不是單純的字串
而是 `Protobuf` 的格式
所以我們在這邊，需要在特別加上一個 `proto.Unmarshal`
稍微改成下面這樣

```go
for {
 d, opened := <-delivery
 if !opened {
  return
 }
 body := pb.FutureRealTimeTickMessage{}
 if err := proto.Unmarshal(d.Body, &body); err != nil {
  continue
 }

 fmt.Printf("%+v\n", body)
}
```

把他反序列化再看看結果

![IMG](https://tocandraw.com/wp-content/uploads/2023/09/【鐵人賽】DAY-17-訂閱即時資料『五』-02.png)

這樣看起來是不舒服多了
至此，已經順利完成送到 `Golang` 這層

## 總結

這段完成以後
接下來需要思考的就是
`Flutter` 跟 `Golang` 的長連線機制
只要能夠再把這些資料順順地送到 `APP`
就可以專心地來設計一個
為自己量身定做的下單介面
