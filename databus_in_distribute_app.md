# 分布式应用中databus的选型和调用

RabbitMQ 是采用 Erlang 语言实现的 AMQP 协议的消息中间件，最初起源于金融系统，用于在分布式系统中存储转发消息。RabbitMQ 发展到今天，被越来越多的人认可，这和它在可靠性、可用性、扩展性、功能丰富等方面的卓越表现是分不开的。

Kafka 起初是由 LinkedIn 公司采用 Scala 语言开发的一个分布式、多分区、多副本且基于 zookeeper 协调的分布式消息系统，现已捐献给 Apache 基金会。它是一种高吞吐量的分布式发布订阅消息系统，以可水平扩展和高吞吐率而被广泛使用。

下面以golang集成kafka为例，描述如何在分布式服务中合理使用消息总线。


- 代码0.0
```
func Init(c *conf.Config, s *service.Service) {
    // 启动tcp协议，启动sarama.Logger，关闭采集 sarama metrics，启动Service
    go accept()
}
```

- 上面的代码，表示服务启动的入口，其中 accept() 协程是一个无限循环的函数。
- 代码0.0.1
```
func accept() {
    for {
        netC, err = listener.Accept()
		go serveConn(netC)
	}
}
```

- accept() 协程启动了一个新的无限循环函数 serveConn(netC) 。
- 代码0.0.1.0
```
// limiter
consumerLimter = make(chan struct{}, 100)
func serveConn(nc net.Conn) {
	c := newConn(nc, _readTimeout, _writeTimeout)
    d, cfg, batch, err = auth(c)
    switch d.Role {
	case _rolePub:
		p, err = NewPub(c, d.Group, d.Topic, d.Color, cfg)
		p.Serve()
	case _roleSub:
		select {
		case consumerLimter <- struct{}{}:
		default:
			err = errCousmerCreateLimiter
            return
        }
		s.Serve()
    default:
    }
}
```

- serveConn(netC) 根据该服务属于kafka client的发布或订阅的角色，启动发布或订阅服务。其中订阅服务的并发数在上面的代码中被设定为100。以下代码以订阅服务为例，调用支持 kafka client 的 sarama-cluster，和 kafka 集群进行交互。
以下是创建一个kafka client的代码。
- 代码0.0.1.0.0
```
import	cluster "github.com/bsm/sarama-cluster"
func NewSub(c *conn, group, topic, color string, sCfg *conf.Kafka, batch int64) (s *Sub, err error) {
	select {
	case <-consumerLimter: //消耗一个并发数
	default:
	}
    type Sub struct {
        consumer    *cluster.Consumer
    }
	s = &Sub{}
    s.consumer, err = cluster.NewConsumer(sCfg.Brokers, group, []string{topic}, cfg)
}
```

- 以下是采用 redis 的命令行参数，执行 kafka sub client 操作的代码，其中有鉴权，心跳，设置KV键值对，读取键值对，丢失连接报错和命令行参数异常报错等功能。
- 代码0.0.1.0.1
```
func (s *Sub) Serve() {
	var (
		err  error
		cmd  string
		args [][]byte
	)
	for {
		cmd, args, err = s.c.Read()
		switch cmd {
		case "auth":
			err = s.write(proto{prefix: '+', message: "OK"})
		case "ping":
			err = s.pong()
		case "mget":
			var enc []byte
			if len(args) > 0 {
				enc = args[0]
			}
			err = s.message(enc)
		case "set":
			err = s.commit(args)
		case "quit":
			err = errors.New("connection closed")
		default:
			err = errors.New("command not support")
		}
		if err != nil {
			log.Error(err)
			return
		}
	}
}
```
- 采用Redis协议，将数据写入kakfa的方案，已经得到多家公司的认同。
> Q：还有就是日志部分现在Redis是瓶颈吗，Redis也是集群？
> A：分享的时候提到了，Redis是瓶颈，后来公司Golang工程师搞了一个Reids--> Kafka的代理服务，采用的Redis协议，但是直接写入到了Kafka，解决了性能问题。
