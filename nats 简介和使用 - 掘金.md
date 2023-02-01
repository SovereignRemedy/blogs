# nats 简介和使用 - 掘金
nats 有 3 个产品

*   core-nats: 不做持久化的及时信息传输系统
*   nats-streaming: 基于 nats 的持久化消息队列(已弃用)
*   nats-jetstream: 基于 nats 的持久化消息队列

这里主要讨论 core-nats 和 nats-jetstream

nats
----

### nats 快速开始

*   启动 nats

```bash

docker run --network host -p 4222:4222 nats

```

*   Connect 连接

```go
nc, err := nats.Connect("nats://localhost:4222")
if err != nil {
  log.Fatal("NATS 连接失败")
}
defer nc.Close()

```

*   Publish 发布/生产消息

```go

err := nc.Publish("foo", []byte("Hello World"))
if err != nil {
  log.Fatal("NATS 发布失败")
}

err = nc.Flush()
if err != nil {
  log.Fatal("NATS Flush 失败")
}

```

出于性能考虑, 发布的消息先写入到类似 Buffer 缓存的地方, 然后再一次性发送到 nats 服务器

参考官方文档: [docs.nats.io/using-nats/…](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.nats.io%2Fusing-nats%2Fdeveloper%2Fsending%2Fcaches "https://docs.nats.io/using-nats/developer/sending/caches")

*   Subscribe 订阅/消费消息

```go

_, err = nc.Subscribe("foo", func(msg *nats.Msg) {
  fmt.Printf("收到消息: %s\n", msg.Data)
})
if err != nil {
  log.Fatal("NATS 订阅失败")
}

```

nats 提供 发布订阅, 请求响应, 和队列模型 3 种 API. 分别是发布订阅模型, 请求响应模型, 和队列模型, 下面展开介绍

### 发布订阅

![](https://markdown-1304103443.cos.ap-guangzhou.myqcloud.com/2022-02-0420220922204545.png)

发布订阅模型, 一个发布者, 多个订阅者, 多个订阅者都可以收到同一个消息

```go

_, err = nc.Subscribe("foo", func(msg *nats.Msg) {
  fmt.Printf("收到消息: %s\n", msg.Data)
})
if err != nil {
  log.Fatal("NATS 订阅失败")
}

```

### 队列模型

![](https://markdown-1304103443.cos.ap-guangzhou.myqcloud.com/2022-02-0420220922204727.png)

队列模型, 一个发布者, 多个订阅者, 消息在多个消息中负载均衡分配, 分配给 A 消费者, 这个消息就不会再分配给其他消费者了

```go


_, _ = nc.QueueSubscribe("foo", "queue", func(msg *nats.Msg) {
	fmt.Printf("收到消息: %s\n", string(msg.Data))
})

```

### 请求响应

生产者能收到消费者的回复

![](https://markdown-1304103443.cos.ap-guangzhou.myqcloud.com/2022-02-0420220922205318.png)

```go

nc.Subscribe("help", func(m *nats.Msg) {
	fmt.Printf("收到消息: %s\n", string(m.Data))
	nc.Publish(m.Reply, []byte("I can help!"))
})


go func() {
	msg, _ := nc.Request("help", []byte("help me"), 100*time.Millisecond)
	fmt.Printf("收到回复: %s\n", string(msg.Data))
}()

select {}

```

#### 应用1-保证消息可靠性

nats 本身不做任何的消息的持久化, 是 "最多一次" 交付模型

> 举个例子, 如果生产的消息没有消费者接, 消息就丢掉了

但是请求响应机制可以通过业务代码保证消息的可靠性, 在业务层面实现常见消息队列的 ACK 机制

> 举个例子, 生产者发送消息, 消费者接受消息后处理, 成功返回 OK, 失败返回 error, 生产者如果收到 error 或者超时就可以补发消息

#### 应用2-解耦 PRC 调用

请求响应模型和 RPC 调用是一致的, 我们可以用这个实现一个基于事件驱动的 RPC 总线

nats-jetstream
--------------

### 架构

jetstream 提供持久化 nats 服务, 客户端支持实时推送的 push 模式和自定义拉取的 pull 模式, 架构图如下

![](https://markdown-1304103443.cos.ap-guangzhou.myqcloud.com/2022-02-0420221008144218.png)

*   subject: 和 nats 一样, 用来区分不同的消息
*   stream: 定义了消息的储存方式, 保留规则, 丢弃规则 (stream 和 subject 是 1:n 的关系)
*   consumer: 定义了消息接受方式并记录接受到的位置, 有 2 种消费方式及时推送 push 和自定义拉取 pull (consumer 和 stream 是 1:n 的关系)

用支持 jetstream 的方式启动 nats

```bash

docker run --network host -p 4222:4222 nats -js

```

### 架构示例

贴一个来自于官方文档的 push pull 混合使用的架构示例图

![](https://markdown-1304103443.cos.ap-guangzhou.myqcloud.com/2022-02-04image-20221008162524128.png)

基于 pull 的 worker 消费者和基于 push 的 monitor 消费者同时存在

### Stream

JetStream 中 Stream 定义了消息的储存方式, 保留规则, 丢弃规则.

一个 Stream 可以对应多个 Subject, 如果一条消息符合 Stream 的保留规则, 就会被保留下来

> 注意 JetStream 所有生产和消费的消息的 Subject 都需要有 Stream 对应, 不然报错

贴一个 Stream 的核心配置 (v1.15.0)

```go




type StreamConfig struct {
    
	Name        string   
    
	Description string
    
	Subjects    []string 
	
	
	
	
	Retention RetentionPolicy 
	
	MaxConsumers int 
	
	MaxMsgs int64 
	
	MaxBytes int64 
	
	
	
	Discard DiscardPolicy 
	
	MaxAge time.Duration 
	
	MaxMsgsPerSubject int64 
	
	MaxMsgSize int32 
	
	Storage StorageType 
	
	Replicas int 
	
	NoAck    bool   
    
}

```

### Consumer

Consumer 定义了消息接受方式并记录接受到的位置

> 举个例子如果消费者在 Sub 消息的时候指定了 Consumer, 就会从记录的位置开始推送消息, 而不是从头开始

贴一个 Consumer 的核心配置 (v1.15.0)

```go


type ConsumerConfig struct {
	
	Durable string `json:"durable_name,omitempty"`
	
	Description string `json:"description,omitempty"`
	
	DeliverSubject string `json:"deliver_subject,omitempty"`
	
	DeliverGroup string `json:"deliver_group,omitempty"`
	
	
	DeliverPolicy DeliverPolicy `json:"deliver_policy"`
	
	OptStartSeq uint64 `json:"opt_start_seq,omitempty"`
	
	OptStartTime *time.Time `json:"opt_start_time,omitempty"`
	
	
	AckPolicy AckPolicy `json:"ack_policy"`
	
	AckWait    time.Duration   `json:"ack_wait,omitempty"`
	MaxDeliver int             `json:"max_deliver,omitempty"`
	BackOff    []time.Duration `json:"backoff,omitempty"`
	
	FilterSubject string `json:"filter_subject,omitempty"`
	
	
	ReplayPolicy ReplayPolicy `json:"replay_policy"`
	
	RateLimit uint64 `json:"rate_limit_bps,omitempty"` 
	
	SampleFrequency string `json:"sample_freq,omitempty"`
	
	MaxWaiting int `json:"max_waiting,omitempty"`
	
	MaxAckPending int `json:"max_ack_pending,omitempty"`
	
	FlowControl bool `json:"flow_control,omitempty"`
	
	Heartbeat time.Duration `json:"idle_heartbeat,omitempty"`
    
}

```

代码示例
----

### Core Publish-Subcribe

```go
package main


import (
	"fmt"
	"os"
	"time"


	"github.com/nats-io/nats.go"
)


func main() {
	
	url := os.Getenv("NATS_URL")
	if url == "" {
		url = nats.DefaultURL
	}

	
	nc, _ := nats.Connect(url)


	defer nc.Drain()

	
	nc.Publish("greet.1", []byte("hello"))

	
	sub, _ := nc.SubscribeSync("greet.*")

	
	msg, _ := sub.NextMsg(10 * time.Millisecond)
	fmt.Println("subscribed after a publish...")
	fmt.Printf("msg is nil? %v\n", msg == nil)

	
	nc.Publish("greet.2", []byte("hello"))
	nc.Publish("greet.3", []byte("hello"))


	msg, _ = sub.NextMsg(10 * time.Millisecond)
	fmt.Printf("msg data: %q on subject %q\n", string(msg.Data), msg.Subject)


	msg, _ = sub.NextMsg(10 * time.Millisecond)
	fmt.Printf("msg data: %q on subject %q\n", string(msg.Data), msg.Subject)


	nc.Publish("greet.4", []byte("hello"))


	msg, _ = sub.NextMsg(10 * time.Millisecond)
	fmt.Printf("msg data: %q on subject %q\n", string(msg.Data), msg.Subject)
}


```

output:

```bash
subscribed after a publish...
msg is nil? true
msg data: "hello" on subject "greet.2"
msg data: "hello" on subject "greet.3"
msg data: "hello" on subject "greet.4"

```

### Request-Reply

```go
package main


import (
	"fmt"
	"os"
	"time"


	"github.com/nats-io/nats.go"
)


func main() {

	url := os.Getenv("NATS_URL")
	if url == "" {
		url = nats.DefaultURL
	}


	nc, _ := nats.Connect(url)
	defer nc.Drain()


	sub, _ := nc.Subscribe("greet.*", func(msg *nats.Msg) {

		name := msg.Subject[6:]
		msg.Respond([]byte("hello, " + name))
	})


	rep, _ := nc.Request("greet.joe", nil, time.Second)
	fmt.Println(string(rep.Data))


	rep, _ = nc.Request("greet.sue", nil, time.Second)
	fmt.Println(string(rep.Data))


	rep, _ = nc.Request("greet.bob", nil, time.Second)
	fmt.Println(string(rep.Data))


	sub.Unsubscribe()


	_, err := nc.Request("greet.joe", nil, time.Second)
	fmt.Println(err)
}

```

output

```bash
hello, joe
hello, sue
hello, bob
nats: no responders available for request

```

### Limits-based Stream

```go
package main


import (
	"encoding/json"
	"fmt"
	"log"
	"os"
	"time"


	"github.com/nats-io/nats.go"
)


func main() {

	url := os.Getenv("NATS_URL")
	if url == "" {
		url = nats.DefaultURL
	}


	nc, _ := nats.Connect(url)


	defer nc.Drain()


	js, _ := nc.JetStream()


	cfg := nats.StreamConfig{
		Name:     "EVENTS",
		Subjects: []string{"events.>"},
	}


	cfg.Storage = nats.FileStorage


	js.AddStream(&cfg)
	fmt.Println("created the stream")


	js.Publish("events.page_loaded", nil)
	js.Publish("events.mouse_clicked", nil)
	js.Publish("events.mouse_clicked", nil)
	js.Publish("events.page_loaded", nil)
	js.Publish("events.mouse_clicked", nil)
	js.Publish("events.input_focused", nil)
	fmt.Println("published 6 messages")


	js.PublishAsync("events.input_changed", nil)
	js.PublishAsync("events.input_blurred", nil)
	js.PublishAsync("events.key_pressed", nil)
	js.PublishAsync("events.input_focused", nil)
	js.PublishAsync("events.input_changed", nil)
	js.PublishAsync("events.input_blurred", nil)


	select {
	case <-js.PublishAsyncComplete():
		fmt.Println("published 6 messages")
	case <-time.After(time.Second):
		log.Fatal("publish took too long")
	}


	printStreamState(js, cfg.Name)


	
	cfg.MaxMsgs = 10
	js.UpdateStream(&cfg)
	fmt.Println("set max messages to 10")


	printStreamState(js, cfg.Name)

	
	cfg.MaxBytes = 300
	js.UpdateStream(&cfg)
	fmt.Println("set max bytes to 300")


	printStreamState(js, cfg.Name)

	
	cfg.MaxAge = time.Second
	js.UpdateStream(&cfg)
	fmt.Println("set max age to one second")


	printStreamState(js, cfg.Name)


	fmt.Println("sleeping one second...")
	time.Sleep(time.Second)


	printStreamState(js, cfg.Name)
}


func printStreamState(js nats.JetStreamContext, name string) {
	info, _ := js.StreamInfo(name)
	b, _ := json.MarshalIndent(info.State, "", " ")
	fmt.Println("inspecting stream info")
	fmt.Println(string(b))
}

```

output

```bash
created the stream
published 6 messages
published 6 messages
inspecting stream info
{
 "messages": 12,
 "bytes": 594,
 "first_seq": 1,
 "first_ts": "2022-07-22T13:04:47.814798969Z",
 "last_seq": 12,
 "last_ts": "2022-07-22T13:04:47.817297637Z",
 "consumer_count": 0
}
set max messages to 10
inspecting stream info
{
 "messages": 10,
 "bytes": 496,
 "first_seq": 3,
 "first_ts": "2022-07-22T13:04:47.815772395Z",
 "last_seq": 12,
 "last_ts": "2022-07-22T13:04:47.817297637Z",
 "consumer_count": 0
}
set max bytes to 300
inspecting stream info
{
 "messages": 6,
 "bytes": 298,
 "first_seq": 7,
 "first_ts": "2022-07-22T13:04:47.817220635Z",
 "last_seq": 12,
 "last_ts": "2022-07-22T13:04:47.817297637Z",
 "consumer_count": 0
}
set max age to one second
inspecting stream info
{
 "messages": 6,
 "bytes": 298,
 "first_seq": 7,
 "first_ts": "2022-07-22T13:04:47.817220635Z",
 "last_seq": 12,
 "last_ts": "2022-07-22T13:04:47.817297637Z",
 "consumer_count": 0
}
sleeping one second...
inspecting stream info
{
 "messages": 0,
 "bytes": 0,
 "first_seq": 13,
 "first_ts": "1970-01-01T00:00:00Z",
 "last_seq": 12,
 "last_ts": "2022-07-22T13:04:47.817297637Z",
 "consumer_count": 0
}

```

更多示例参考: [natsbyexample.com/](https://link.juejin.cn/?target=https%3A%2F%2Fnatsbyexample.com%2F "https://natsbyexample.com/")

reference
---------

官方文档: [docs.nats.io/](https://link.juejin.cn/?target=https%3A%2F%2Fdocs.nats.io%2F "https://docs.nats.io/")

官方GitHub: [github.com/nats-io/nat…](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fnats-io%2Fnats.go "https://github.com/nats-io/nats.go")

代码示例: [natsbyexample.com/](https://link.juejin.cn/?target=https%3A%2F%2Fnatsbyexample.com%2F "https://natsbyexample.com/")

[marco79423.net/articles/%E…](https://link.juejin.cn/?target=https%3A%2F%2Fmarco79423.net%2Farticles%2F%25E6%25B7%25BA%25E8%25AB%2587-natsstan-%25E5%2592%258C-jetstream-%25E5%2585%25A9%25E4%25B8%2589%25E4%25BA%258B "https://marco79423.net/articles/%E6%B7%BA%E8%AB%87-natsstan-%E5%92%8C-jetstream-%E5%85%A9%E4%B8%89%E4%BA%8B")

本文由[mdnice](https://link.juejin.cn/?target=https%3A%2F%2Fmdnice.com%2F%3Fplatform%3D2 "https://mdnice.com/?platform=2")多平台发布