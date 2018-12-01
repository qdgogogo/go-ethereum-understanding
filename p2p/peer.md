# peer #

----------
peer 

代表已建立连接的远程节点

结构


	type Peer struct {
		rw      *conn //用两次握手期间收集的信息包装网络连接。
		running map[string]*protoRW //运行的协议
		log     log.Logger
		created mclock.AbsTime
	
		wg       sync.WaitGroup
		protoErr chan error
		closed   chan struct{}
		disc     chan DiscReason
	
		// 接收消息发送/接收事件（如果已设置）
		events *event.Feed
	}

protoRW 结构

	type protoRW struct {
		Protocol
		in     chan Msg        // 读取消息的通道
		closed <-chan struct{} // 接受关闭消息
		wstart <-chan struct{} // 当写操作开始时接受
		werr   chan<- error    // 
		offset uint64
		w      MsgWriter		//发送消息
	}

Protocol 的结构

代表P2P子协议实现

	type Protocol struct {
		// 应包含官方协议名称，通常是三个字母的单词。
		Name string
	
		// 版本
		Version uint
	
		// 消息的长度
		Length uint64
	

		//开启新的goroutine，从rw读取和写入消息，每条消息的负载必须满额。
		//
		//当start方法返回时peer的链接被关闭，应当返回任何协议级的错误例如i/o错误。
		Run func(peer *Peer, rw MsgReadWriter) error
	
		// 一个可选的辅助方法，用于检索有关主机节点的协议特定元数据。
		NodeInfo func() interface{}
	
		// 一种可选的辅助方法，用于检索有关网络中某个对等方的协议特定元数据。 如果设置了信息检索功能，但返回nil，则认为协议握手仍在运行。
		PeerInfo func(id discover.NodeID) interface{}
	}

1.1 newPeer

peer的创建
根据匹配找到当前Peer支持的protomap

	func newPeer(conn *conn, protocols []Protocol) *Peer {
		protomap := matchProtocols(protocols, conn.caps, conn)
		p := &Peer{
			rw:       conn,
			running:  protomap,
			created:  mclock.Now(),
			disc:     make(chan DiscReason),
			protoErr: make(chan error, len(protomap)+1), // protocols + pingLoop
			closed:   make(chan struct{}),
			log:      log.New("id", conn.node.ID(), "conn", conn.flags),
		}
		return p
	}

1.1.1 matchProtocols

创建用于匹配命名子协议的结构。

最终每个子协议以name=>protocol的map格式组织起来,然后每个协议根据自身支持消息类型数量Protocol.Length在整个以太坊消息类型轴上占据了[proto.offset,proto.offset+proto.Length)的左闭右开消息类型段,理解这个结构,才好理解最终根据消息类型Msg.Code去找handler的逻辑(func (p *Peer) getProto(code uint64) (*protoRW, error))。
	
	func matchProtocols(protocols []Protocol, caps []Cap, rw MsgReadWriter) map[string]*protoRW {
		sort.Sort(capsByNameAndVersion(caps)) //根据名称和版本排序
		
		//协议偏移？
		 //目前不知道这是做什么的，只知道根据协议的的大小对应-todo
		offset := baseProtocolLength  
		
		result := make(map[string]*protoRW)
	
	outer:
		for _, cap := range caps {
			for _, proto := range protocols {
				if proto.Name == cap.Name && proto.Version == cap.Version {
					// 如果旧协议版本匹配，则还原它
					if old := result[cap.Name]; old != nil {
						offset -= old.Length
					}
					// 分配新匹配
					result[cap.Name] = &protoRW{Protocol: proto, offset: offset, in: make(chan Msg), w: rw}
					offset += proto.Length
	
					continue outer
				}
			}
		}
		return result
	}



2.1 peer的run方法

	func (p *Peer) run() (remoteRequested bool, err error) {
		var (
			writeStart = make(chan struct{}, 1)
			writeErr   = make(chan error, 1)
			readErr    = make(chan error, 1)
			reason     DiscReason // sent to the peer
		)
		p.wg.Add(2) //将下面两个goroutine添加到等待组中
		go p.readLoop(readErr)
		go p.pingLoop()
	
		// 启动所有协议处理程序。
		writeStart <- struct{}{}
		p.startProtocols(writeStart, writeErr)
	
		// 等待错误或断开连接。
		//处理错误
	loop:
		for {
			select {
			case err = <-writeErr:
				// A write finished. Allow the next write to start if
				// there was no error.
				// 上一个写操作结束，如果没有错误下一个写将开始
				if err != nil {
					reason = DiscNetworkError
					break loop
				}
				writeStart <- struct{}{}
			case err = <-readErr:
				if r, ok := err.(DiscReason); ok {
					remoteRequested = true
					reason = r
				} else {
					reason = DiscNetworkError
				}
				break loop
			case err = <-p.protoErr:
				reason = discReasonForError(err)
				break loop
			case err = <-p.disc:
				reason = discReasonForError(err)
				break loop
			}
		}
	
		close(p.closed)
		p.rw.close(reason)
		p.wg.Wait()
		return remoteRequested, err
	}

2.1.1 readLoop

	func (p *Peer) readLoop(errc chan<- error) {
		defer p.wg.Done()
		for {
			//读取消息
			msg, err := p.rw.ReadMsg()
			if err != nil {
				errc <- err
				return
			}
			msg.ReceivedAt = time.Now()
			//消息处理
			if err = p.handle(msg); err != nil {
				errc <- err
				return
			}
		}
	}

2.1.1.1 ReadMsg

	func (rw *protoRW) ReadMsg() (Msg, error) {
		select {
		case msg := <-rw.in:
			msg.Code -= rw.offset  //这里的offset
			return msg, nil
		case <-rw.closed:
			return Msg{}, io.EOF
		}
	}

2.1.1.2 handle

读取消息之后，通过handle()函数来进行消息的不同类型的相应的处理。

	func (p *Peer) handle(msg Msg) error {
		switch {
		case msg.Code == pingMsg:  //ping消息
			msg.Discard()
			go SendItems(p.rw, pongMsg)
		case msg.Code == discMsg:
			var reason [1]DiscReason
			// This is the last message. We don't need to discard or
			// check errors because, the connection will be closed after it.
			// 最后一条消息，我们不需要检查，应为链接将会关闭
			rlp.Decode(msg.Payload, &reason)
			return reason[0]
		case msg.Code < baseProtocolLength:
			// 忽略其他基础协议得消息
			return msg.Discard()
		default:
			// 通过偏移量offset和长度Length取出对应协议对应msg.Code
			// 子协议消息
			proto, err := p.getProto(msg.Code)
			if err != nil {
				return fmt.Errorf("msg code out of range: %v", msg.Code)
			}
			select {
			case proto.in <- msg:
				return nil
			case <-p.closed:
				return io.EOF
			}
		}
		return nil
	}

2.1.1.2.1 getProto

用给出的code去找到具体的协议

	func (p *Peer) getProto(code uint64) (*protoRW, error) {
		for _, proto := range p.running {
			if code >= proto.offset && code < proto.offset+proto.Length {
				return proto, nil
			}
		}
		return nil, newPeerError(errInvalidMsgCode, "%d", code)
	}


2.1.2 pingLoop

每隔15秒发送ping消息

	func (p *Peer) pingLoop() {
		ping := time.NewTimer(pingInterval)
		defer p.wg.Done()
		defer ping.Stop()
		for {
			select {
			case <-ping.C:
				if err := SendItems(p.rw, pingMsg); err != nil {  //不知道给谁发送ping-todo
					p.protoErr <- err
					return
				}
				ping.Reset(pingInterval)
			case <-p.closed:
				return
			}
		}
	}

2.1.3 startProtocols

遍历执行每个协议的Run方法

	func (p *Peer) startProtocols(writeStart <-chan struct{}, writeErr chan<- error) {
		p.wg.Add(len(p.running)) //添加到等待队列
		for _, proto := range p.running {
			proto := proto
			proto.closed = p.closed
			proto.wstart = writeStart
			proto.werr = writeErr
			var rw MsgReadWriter = proto
			if p.events != nil {
				//返回一个msgEventer，它将消息事件发送到给定的feed
				rw = newMsgEventer(rw, p.events, p.ID(), proto.Name)
			}
			p.log.Trace(fmt.Sprintf("Starting protocol %s/%d", proto.Name, proto.Version))
			go func() {
				//目前不清楚run方法的具体细节-todo
				err := proto.Run(p, rw)
				if err == nil {
					p.log.Trace(fmt.Sprintf("Protocol %s/%d returned", proto.Name, proto.Version))
					err = errProtocolReturned
				} else if err != io.EOF {
					p.log.Trace(fmt.Sprintf("Protocol %s/%d failed", proto.Name, proto.Version), "err", err)
				}
				p.protoErr <- err
				p.wg.Done()
			}()
		}
	}

2.2 WriteMsg和ReadMsg


在写消息时msg.Code += rw.offset，读消息时msg.Code -= rw.offset。
	
	func (rw *protoRW) WriteMsg(msg Msg) (err error) {
		if msg.Code >= rw.Length {
			return newPeerError(errInvalidMsgCode, "not handled")
		}
		msg.Code += rw.offset
		select {
		case <-rw.wstart:
			err = rw.w.WriteMsg(msg)
			// Report write status back to Peer.run. It will initiate
			// shutdown if the error is non-nil and unblock the next write
			// otherwise. The calling protocol code should exit for errors
			// as well but we don't want to rely on that.
			rw.werr <- err
		case <-rw.closed:
			err = ErrShuttingDown
		}
		return err
	}
	
	func (rw *protoRW) ReadMsg() (Msg, error) {
		select {
		case msg := <-rw.in:
			msg.Code -= rw.offset
			return msg, nil
		case <-rw.closed:
			return Msg{}, io.EOF
		}
	}