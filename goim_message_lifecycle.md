# goim的一条消息的生命周期

- 不包括对服务注册/发现和负载均衡的分析。

## logic 用 kafka client 发布通信内容。

- cmd/logic/main.go
```
func main() {
	if err := conf.Init(); err != nil {
		panic(err)
	}
	srv := logic.New(conf.Conf)
	httpSrv := http.New(conf.Conf.HTTPServer, srv)
}
```



```
// New new a http server.
func New(c *conf.HTTPServer, l *logic.Logic) *Server {
	s := &Server{
		engine: engine,
		logic:  l,
	}
	s.initRouter()
	return s
}

func (s *Server) initRouter() {
	group := s.engine.Group("/goim")
	group.POST("/push/keys", s.pushKeys)
}
```



```
func (s *Server) pushKeys(c *gin.Context) {
	err := c.BindQuery(&arg)
	msg, err := ioutil.ReadAll(c.Request.Body)
	err = s.logic.PushKeys(context.TODO(), arg.Op, arg.Keys, msg)
	result(c, nil, OK)
}
```



```
// PushKeys push a message by keys.
func (l *Logic) PushKeys(c context.Context, op int32, keys []string, msg []byte) (err error) {
	for server := range pushKeys {
		if err = l.dao.PushMsg(c, op, server, pushKeys[server], msg); err != nil {
			return
		}
	}
	return
}
```



```
// PushMsg push a message to databus.
func (d *Dao) PushMsg(c context.Context, op int32, server string, keys []string, msg []byte) (err error) {
	m := &sarama.ProducerMessage{
		Key:   sarama.StringEncoder(keys[0]),
		Topic: d.c.Kafka.Topic,
		Value: sarama.ByteEncoder(b),
	}
    _, _, err = d.kafkaPub.SendMessage(m)
	return
}
```

## job 通过 kafka 订阅 logic 发布的通信内容。

- cmd/job/main.go
```
func main() {
	j := job.New(conf.Conf)
	go j.Consume()
}
```

```
// Job is push job.
type Job struct {
	c            *conf.Config
	consumer     *cluster.Consumer
	cometServers map[string]*Comet

	rooms      map[string]*Room
	roomsMutex sync.RWMutex
}

// Consume messages, watch signals
func (j *Job) Consume() {
	for {
		select {
		case msg, ok := <-j.consumer.Messages():
			if !ok {
				return
			}
			pushMsg := new(pb.PushMsg)
			err := proto.Unmarshal(msg.Value, pushMsg)
			if err := j.push(context.Background(), pushMsg); err != nil {
				log.Errorf("j.push(%v) error(%v)", pushMsg, err)
			}
		}
	}
}
```



```
func (j *Job) push(ctx context.Context, pushMsg *pb.PushMsg) (err error) {
	switch pushMsg.Type {
	case pb.PushMsg_PUSH:
		err = j.pushKeys(pushMsg.Operation, pushMsg.Server, pushMsg.Keys, pushMsg.Msg)
	}
	return
}

// pushKeys push a message to a batch of subkeys.
func (j *Job) pushKeys(operation int32, serverID string, subKeys []string, body []byte) (err error) {
	buf := bytes.NewWriterSize(len(body) + 64)
	p := &comet.Proto{
		Ver:  1,
		Op:   operation,
		Body: body,
	}
	p.WriteTo(buf)
	p.Body = buf.Buffer()
	p.Op = comet.OpRaw
	var args = comet.PushMsgReq{
		Keys:    subKeys,
		ProtoOp: operation,
		Proto:   p,
	}
	if c, ok := j.cometServers[serverID]; ok {
		if err = c.Push(&args); err != nil {
			log.Errorf("c.Push(%v) serverID:%s error(%v)", args, serverID, err)
		}
		log.Infof("pushKey:%s comets:%d", serverID, len(j.cometServers))
	}
	return
}
```



```
// Comet is a comet.
type Comet struct {
	serverID      string
	client        comet.CometClient
	pushChan      []chan *comet.PushMsgReq
	roomChan      []chan *comet.BroadcastRoomReq
	broadcastChan chan *comet.BroadcastReq
	pushChanNum   uint64
	roomChanNum   uint64
	routineSize   uint64

	ctx    context.Context
	cancel context.CancelFunc
}

// Push push a user message.
func (c *Comet) Push(arg *comet.PushMsgReq) (err error) {
	idx := atomic.AddUint64(&c.pushChanNum, 1) % c.routineSize
	c.pushChan[idx] <- arg
	return
}

// NewComet new a comet.
func NewComet(in *naming.Instance, c *conf.Comet) (*Comet, error) {
	for i := 0; i < c.RoutineSize; i++ {
		cmt.pushChan[i] = make(chan *comet.PushMsgReq, c.RoutineChan)
		cmt.roomChan[i] = make(chan *comet.BroadcastRoomReq, c.RoutineChan)
		go cmt.process(cmt.pushChan[i], cmt.roomChan[i], cmt.broadcastChan)
	}
	return cmt, nil
}

// Push push a user message.
func (c *Comet) Push(arg *comet.PushMsgReq) (err error) {
	idx := atomic.AddUint64(&c.pushChanNum, 1) % c.routineSize
	c.pushChan[idx] <- arg
	return
}

func (c *Comet) process(pushChan chan *comet.PushMsgReq, roomChan chan *comet.BroadcastRoomReq, broadcastChan chan *comet.BroadcastReq) {
	for {
		select {
		case pushArg := <-pushChan:
			_, err := c.client.PushMsg(context.Background(), &comet.PushMsgReq{
				Keys:    pushArg.Keys,
				Proto:   pushArg.Proto,
				ProtoOp: pushArg.ProtoOp,
			})
		case <-c.ctx.Done():
			return
		}
	}
}
```


## comet 用 grpc 协议接收 job 服务发送的通信内容。

- internal/comet/grpc/server.go
```
// PushMsg push a message to specified sub keys.
func (s *server) PushMsg(ctx context.Context, req *pb.PushMsgReq) (reply *pb.PushMsgReply, err error) {
	if len(req.Keys) == 0 || req.Proto == nil {
		return nil, errors.ErrPushMsgArg
	}
	for _, key := range req.Keys {
		if channel := s.srv.Bucket(key).Channel(key); channel != nil {
			if !channel.NeedPush(req.ProtoOp) {
			if err = channel.Push(req.Proto); err != nil {
				return
			}
		}
	}
	return &pb.PushMsgReply{}, nil
}
```

- internal/comet/channel.go
```
// Push server push message.
func (c *Channel) Push(p *grpc.Proto) (err error) {
	select {
	case c.signal <- p:
	default:
	}
	return
}

// Ready check the channel ready or close?
func (c *Channel) Ready() *grpc.Proto {
	return <-c.signal
}
```

- 通过 grpc 连接到 logic ，用 (*grpc.Proto).WriteTCP(wr)的方法，传递聊天的内容。
- internal/comet/server_tcp.go
```

// dispatch accepts connections on the listener and serves requests
// for each incoming connection.  dispatch blocks; the caller typically
// invokes it in a go statement.
func (s *Server) dispatchTCP(conn *net.TCPConn, wr *bufio.Writer, wp *bytes.Pool, wb *bytes.Buffer, ch *Channel) {
	for {
		var p = ch.Ready()
		switch p {
		default:
			// server send
			if err = p.WriteTCP(wr); err != nil {
			}
		}
	}
}
```

- api/comet/grpc/protocol.go
```
// WriteTCP write a proto to TCP writer.
func (p *Proto) WriteTCP(wr *bufio.Writer) (err error) {
	var (
		buf     []byte
		packLen int32
	)
	if p.Op == OpRaw {
		// write without buffer, job concact proto into raw buffer
		_, err = wr.WriteRaw(p.Body)
		return
	}
	packLen = _rawHeaderSize + int32(len(p.Body))
	if buf, err = wr.Peek(_rawHeaderSize); err != nil {
		return
	}
	binary.BigEndian.PutInt32(buf[_packOffset:], packLen)
	binary.BigEndian.PutInt16(buf[_headerOffset:], int16(_rawHeaderSize))
	binary.BigEndian.PutInt16(buf[_verOffset:], int16(p.Ver))
	binary.BigEndian.PutInt32(buf[_opOffset:], p.Op)
	binary.BigEndian.PutInt32(buf[_seqOffset:], p.Seq)
	if p.Body != nil {
		_, err = wr.Write(p.Body)
	}
	return
}
```