# udp协议 #

----------


Kademlia protocol使用了UDP协议来进行网络通信。

以下是我个人通过源码对以太坊udp的一些理解


## 结构体定义 ##

首先是rpc请求结构体，包括ping，pong，findnode等

    type (
    	ping struct {
    		Version		uint  //版本号
    		From, To   rpcEndpoint //从哪到哪
    		Expiration uint64 	//时间戳
    		// 向前兼容，这里有待考究-todo
    		Rest []rlp.RawValue `rlp:"tail"`
    	}
    
    	// pong is the reply to ping.
		//对ping的回应
    	pong struct {
    		// This field should mirror the UDP envelope address
    		// of the ping packet, which provides a way to discover the
    		// the external address (after NAT).
    		To rpcEndpoint	//目的地址
    
    		ReplyTok   []byte // ping的哈希，为了说明是回应的具体哪个ping包
    		Expiration uint64 
    		Rest []rlp.RawValue `rlp:"tail"`
    	}
    
    	// 查找距离目标节点近的节点
    	findnode struct {
    		Target NodeID // 目标节点
    		Expiration uint64
    		Rest []rlp.RawValue `rlp:"tail"`
    	}
    
    	// 对findnode的回应
    	neighbors struct {
    		Nodes  []rpcNode 	//查找到的节点
    		Expiration uint64
    		// Ignore additional fields (for forward compatibility).
    		Rest []rlp.RawValue `rlp:"tail"`
    	}
    
    	rpcNode struct {
    		IP  net.IP // len 4 for IPv4 or 16 for IPv6
    		UDP uint16 // for discovery protocol
    		TCP uint16 // for RLPx protocol
    		ID  NodeID
    	}
    
    	rpcEndpoint struct {
    		IP  net.IP // len 4 for IPv4 or 16 for IPv6
    		UDP uint16 // for discovery protocol
    		TCP uint16 // for RLPx protocol
    	}
    )

定义了两个接口类型

	//4中结构体都继承了该接口，实现了不同的handle方法
    type packet interface {
    	handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error
    	name() string
    }
    //定义了udp链接的读写等功能
    type conn interface {
    	ReadFromUDP(b []byte) (n int, addr *net.UDPAddr, err error)
    	WriteToUDP(b []byte, addr *net.UDPAddr) (n int, err error)
    	Close() error
    	LocalAddr() net.Addr
    }

pending结构体

pending表示待处理的回复。协议的一些实现希望向findnode发送多个回复包。 通常，任何neighbors包都不能与特定的findnode包匹配。
我们的实现通过为每个待处理的回复存储回调函数来处理这个问题。 来自节点的传入数据包被分派到该节点的所有回调函数。

	type pending struct {
	// 这些必须在回复中匹配
	from  NodeID
	ptype byte

	// 请求必须完成的时间
	deadline time.Time


	//当匹配的收到回复该函数被调用，如果返回值是真则加入到回复队列，如果为假则为匹配到或者没有回复
	callback func(resp interface{}) (done bool)

	//当回调表示完成时，errc收到nil，如果超时内没有收到进一步的回复，则收到错误。
	errc chan<- error
	}
   
udp的结构体，udp实现了RPC协议


    type udp struct {
	conn        conn  //
	netrestrict *netutil.Netlist //ip网络的列表
	priv        *ecdsa.PrivateKey	//私钥
	ourEndpoint rpcEndpoint		//自身的rpc

	addpending chan *pending	//申请一个pending
	gotreply   chan reply		//获取回应的队列

	closing chan struct{}		//关闭的通道
	nat     nat.Interface		

	*Table			//go里面的匿名字段。 也就是说udp可以直接调用匿名字段Table的方法。
	}
 
reply 回复的结构体

    type reply struct {
	from  NodeID	//回复节点的ID
	ptype byte		//类型？-todo
	data  interface{}	//回复的数据
	// 一旦有匹配的回复将会发送到这个通道
	matched chan<- bool		
	}




## udp创建 ##


ListenUDP返回一个新表，用于监听laddr上的UDP数据包.这里调用了newUDP。
newUDP中初始化自身的表，具体的内容在[table](https://github.com/qdgogogo/go-ethereum-understanding/blob/master/table.md)中讲到。

1.

	func ListenUDP(c conn, cfg Config) (*Table, error) {
		tab, _, err := newUDP(c, cfg)
		if err != nil {
			return nil, err
		}
		log.Info("UDP listener up", "self", tab.self)
		return tab, nil
	}

	func newUDP(c conn, cfg Config) (*Table, *udp, error) {
		udp := &udp{
			conn:        c,
			priv:        cfg.PrivateKey,
			netrestrict: cfg.NetRestrict,//白名单
			closing:     make(chan struct{}),
			gotreply:    make(chan reply),
			addpending:  make(chan *pending),
		}
		realaddr := c.LocalAddr().(*net.UDPAddr)//括号中的UDPAddr是对net.addr的实现
		if cfg.AnnounceAddr != nil {
			realaddr = cfg.AnnounceAddr
		}
		// TODO: separate TCP port
		//初始化自身的rpc节点信息，这里将realaddr作为自身地址
		udp.ourEndpoint = makeEndpoint(realaddr, uint16(realaddr.Port))
		//初始化表，PubkeyID用公钥计算nodeID
		tab, err := newTable(udp, PubkeyID(&cfg.PrivateKey.PublicKey), realaddr, cfg.NodeDBPath, cfg.Bootnodes)
		if err != nil {
			return nil, nil, err
		}
		udp.Table = tab
		// 开启循环，跟踪刷新计时器和待处理的回复队列。
		go udp.loop()
		go udp.readLoop(cfg.Unhandled)  //处理收到的数据
		return udp.Table, udp, nil
	}
1.1 

udp主要发送两种请求，对应的也会接收别人发送的这两种请求， 对应这两种请求又会产生两种回应。


ping
向给定节点发送消息，并等待回复

	func (t *udp) ping(toid NodeID, toaddr *net.UDPAddr) error {
		req := &ping{
			Version:    Version,
			From:       t.ourEndpoint,
			To:         makeEndpoint(toaddr, 0), // TODO: maybe use known TCP port from DB
			Expiration: uint64(time.Now().Add(expiration).Unix()), //20秒过期时间
		}
		packet, hash, err := encodePacket(t.priv, pingPacket, req)
		if err != nil {
			return err
		}
		//希望得到一个pong回复
		errc := t.pending(toid, pongPacket, func(p interface{}) bool {
			return bytes.Equal(p.(*pong).ReplyTok, hash)  //这里的回调方法是判断pong中回复的hash是否与ping的数据包哈希一致
		})
		t.write(toaddr, req.name(), packet)
		return <-errc
	}
1.1.2 encodePacket
对数据包加密

	func encodePacket(priv *ecdsa.PrivateKey, ptype byte, req interface{}) (packet, hash []byte, err error) {
		b := new(bytes.Buffer)
		b.Write(headSpace)
		b.WriteByte(ptype) //这里的类型是指ping还是pong
		if err := rlp.Encode(b, req); err != nil { //尝试rlp编码
			log.Error("Can't encode discv4 packet", "err", err)
			return nil, nil, err
		}
		packet = b.Bytes()
		sig, err := crypto.Sign(crypto.Keccak256(packet[headSize:]), priv) //对出去头数据，也就是类型和req数据签名
		if err != nil {
			log.Error("Can't sign discv4 packet", "err", err)
			return nil, nil, err
		}
		copy(packet[macSize:], sig)  //将签名复制到packet，不明白这里为什么只留macSize-todo
	
		hash = crypto.Keccak256(packet[macSize:])//计算哈希
		copy(packet, hash)		//将哈希添加到packet前面，注意这里返回的是两个值
		return packet, hash, nil
	}


1.1.2pending
pending将回复的回调添加到待处理的回复队列

	func (t *udp) pending(id NodeID, ptype byte, callback func(interface{}) bool) <-chan error {
		ch := make(chan error, 1)
		p := &pending{from: id, ptype: ptype, callback: callback, errc: ch}
		select {
		case t.addpending <- p:
			// loop will handle it
		case <-t.closing:
			ch <- errClosed
		}
		return ch
	}

1.1.3 ping请求的处理

	func (req *ping) handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error {
		if expired(req.Expiration) { //判断请求时间的合理性，也就是请求必须发生在当前时间之前
			return errExpired
		}
		//收到别的节点发送的ping请求，发送pong回答
		t.send(from, pongPacket, &pong{  //pong回应
			To:         makeEndpoint(from, req.From.TCP),
			ReplyTok:   mac,
			Expiration: uint64(time.Now().Add(expiration).Unix()),
		})
		//这里handleReply是验证与之匹配的请求是否在pending中，但是这是对ping的回复，可能自己是被请求的一方，所以在pending中没有请求的记录，所以这里它做出了bond处理，也就是自己被ping后和对方建立链接
		if !t.handleReply(fromID, pingPacket, req) { //判断回复通道是否关闭
			// Note: we're ignoring the provided IP address right now
			//建立绑定
			go t.bond(true, fromID, from, req.From.TCP)
		}
		return nil
	}


1.1.4 pong请求的处理
	
	func (req *pong) handle(t *udp, from *net.UDPAddr, fromID NodeID, mac []byte) error {
		if expired(req.Expiration) { //判断是否超时
			return errExpired
		}
		if !t.handleReply(fromID, pongPacket, req) { 判断是否与pending中的匹配
			return errUnsolicitedReply
		}
		return nil
	}




1.2 loop()
开启循环，跟踪刷新计时器和待处理的回复队列。

    func (t *udp) loop() {
		var (
			plist        = list.New()  //创建一个双向链表
			timeout      = time.NewTimer(0) //到期时间0，是什么意思
			nextTimeout  *pending // plist的头部
			contTimeouts = 0      // 进行NTP检查的连续超时次数
			ntpWarnTime  = time.Unix(0, 0) //根据所提供的两个参数然会本地时间
		)
		在go语言中类似于<-timeout.C，通道左边没有接受对象的默认是将通道中的对象丢弃吗？-todo
		<-timeout.C // 忽略第一次到期
		defer timeout.Stop()
	
		resetTimeout := func() {
			//链表的第一个值为空或者值为nextTimeout
			if plist.Front() == nil || nextTimeout == plist.Front().Value {
				return
			}
			// Start the timer so it fires when the next pending reply has expired.
			//启动计时器，以便在下一个pending reply到期时触发
			now := time.Now()
			//遍历链表
			for el := plist.Front(); el != nil; el = el.Next() {
				nextTimeout = el.Value.(*pending)
				//这里deadline可以理解为请求的最后期限
				//所以这里dist为现在距离请求的最后期限还有多久，如果小于1秒
				//不太懂-todo
				if dist := nextTimeout.deadline.Sub(now); dist < 2*respTimeout {
					//重新开始计时，设置过期时间为dist
					timeout.Reset(dist)
					return
				}
				//删除未来截止日期过长的pending replies。 如果系统时钟在分配截止日期后向后跳跃，则会发生这种情况。
				//这种情况在修改系统时间的时候有可能发生，如果不处理可能导致堵塞太长时间。
				// Remove pending replies whose deadline is too far in the
				// future. These can occur if the system clock jumped
				// backwards after the deadline was assigned.
				nextTimeout.errc <- errClockWarp
				plist.Remove(el)
			}
			nextTimeout = nil
			timeout.Stop()
		}
	
		for {
			resetTimeout() //处理超时
	
			select {
	
			case <-t.closing://收到关闭信息。 超时所有的堵塞的队列
				for el := plist.Front(); el != nil; el = el.Next() {
					el.Value.(*pending).errc <- errClosed
				}
				return
			//
			case p := <-t.addpending: //设置回复超时，0.5秒	
				p.deadline = time.Now().Add(respTimeout)
				plist.PushBack(p) 在链表的后面插入元素p
	
			//4种请求处理都会被回复
			case r := <-t.gotreply:
				
				var matched bool
				for el := plist.Front(); el != nil; el = el.Next() {
					p := el.Value.(*pending)
					if p.from == r.from && p.ptype == r.ptype { //如果回复与pending匹配
						matched = true
						//如果其回调表明已收到所有回复，请删除匹配器。 对于期望多个回复数据包的数据包类型，这是必需的。
						//如果没有完成会继续等待下一次reply.
						if p.callback(r.data) {//如果回复的数据满足pending的回调函数，则该pending完成，从链表中删除
							p.errc <- nil
							plist.Remove(el)
						}
						// 重置超时计数器？-todo
						contTimeouts = 0
					}
				}
				r.matched <- matched
	
			case now := <-timeout.C:
				nextTimeout = nil
	
				// Notify and remove callbacks whose deadline is in the past.
				for el := plist.Front(); el != nil; el = el.Next() {
					p := el.Value.(*pending)
					if now.After(p.deadline) || now.Equal(p.deadline) {//如果超时写入超时信息并移除
						p.errc <- errTimeout
						plist.Remove(el)
						contTimeouts++
					}
				}
				// //如果连续超时很多次。 那么查看是否是时间不同步。 和NTP服务器进行同步。
				if contTimeouts > ntpFailureThreshold {
					if time.Since(ntpWarnTime) >= ntpWarningCooldown {
						ntpWarnTime = time.Now()
						go checkClockDrift()
					}
					contTimeouts = 0
				}
			}
		}
	}






1.3 readLoop
在自己的goroutine中运行。 它处理传入的UDP数据包。

	func (t *udp) readLoop(unhandled chan<- ReadPacket) {
		defer t.conn.Close()
		if unhandled != nil {// 在结束时关闭未处理包的通道
			defer close(unhandled)
		}
		//发现数据包被定义为不大于1280字节。 大于此大小的数据包将在末尾被剪切并视为无效，因为它们的哈希值不匹配。
		buf := make([]byte, 1280)
		for {
			nbytes, from, err := t.conn.ReadFromUDP(buf)
			if netutil.IsTemporaryError(err) {//检查给定的错误是否应该被视为临时错误。
				// Ignore temporary read errors.
				log.Debug("Temporary UDP read error", "err", err)
				continue
			} else if err != nil {
				// Shut down the loop for permament errors.
				log.Debug("UDP read error", "err", err)
				return
			}
			if t.handlePacket(from, buf[:nbytes]) != nil && unhandled != nil {
				select {
				case unhandled <- ReadPacket{buf[:nbytes], from}: //目前不清楚未处理的数据？-todo
				default:
				}
			}
		}
	}

1.3.1 handlePacket
处理数据包

	func (t *udp) handlePacket(from *net.UDPAddr, buf []byte) error {
		packet, fromID, hash, err := decodePacket(buf) //解码
		if err != nil {
			log.Debug("Bad discv4 packet", "addr", from, "err", err)
			return err
		}	
		//这里handle实现了4种数据类型的handle
		err = packet.handle(t, from, fromID, hash)
		log.Trace("<< "+packet.name(), "addr", from, "err", err)
		return err
	}

1.3.2 decodePacket
具体的解码过程
	
	func decodePacket(buf []byte) (packet, NodeID, []byte, error) {
		if len(buf) < headSize+1 { //如果长度不过数据为空
			return nil, NodeID{}, nil, errPacketTooSmall
		}
		//也就是说数据包的macSize大小的hash，[macSize:headSize]的签名，headSize之后的具体数据
		hash, sig, sigdata := buf[:macSize], buf[macSize:headSize], buf[headSize:]
		shouldhash := crypto.Keccak256(buf[macSize:])
		if !bytes.Equal(hash, shouldhash) { //验证以下hash
			return nil, NodeID{}, nil, errBadHash
		}
		//验证签名并计算ID
		fromID, err := recoverNodeID(crypto.Keccak256(buf[headSize:]), sig)
		if err != nil {
			return nil, NodeID{}, hash, err
		}
		var req packet
		//数据的第一个字节是数据的类型，总共有4种类型
		//ping,pong,findnode,neighbors
		switch ptype := sigdata[0]; ptype {
		case pingPacket:
			req = new(ping)
		case pongPacket:
			req = new(pong)
		case findnodePacket:
			req = new(findnode)
		case neighborsPacket:
			req = new(neighbors)
		default:
			return nil, fromID, hash, fmt.Errorf("unknown type: %d", ptype)
		}
		s := rlp.NewStream(bytes.NewReader(sigdata[1:]), 0)
		err = s.Decode(req)
		return req, fromID, hash, err
	}





总结

pending是一个待回复的请求链表，接受的类型有Ping请求和findnode请求。以Ping为例，当节点要发送ping请求前先将它加入pending，在pending中它期望得到的是Pong回复，然后发送具体的请求数据，等待接受消息。对应的，如果对方接收到ping请求，就会有上述1.1.3中对ping请求的处理，也就是发送pong回复。随后请求发送方会收到pong消息，进行处理1.1.4，判断是否与之前加入到pending中的请求匹配，如果匹配，并且数据接受完毕则从pending中删除请求。总的来说loop是处理请求的一个循环，例如删除超时请求等，readloop是读取请求解码并让对应的处理方法处理，然后再发送消息到gotreply让loop处理，两个循环是相互作用的。