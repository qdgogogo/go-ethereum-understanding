# server #

----------
P2P启动位于p2p模块server.go, 初始化的工作放在Start()方法中。

具体的流程可以在[这里](https://blog.csdn.net/niyuelin1990/article/details/80195974)看到。


Config

server的配置选项

	type Config struct {
		// 必须是有效的secp256k1 私钥
		PrivateKey *ecdsa.PrivateKey `toml:"-"`
	
		// MaxPeers 是可连接的最大peer数
		MaxPeers int
	
		// 握手阶段可以挂起的最大peer数，分别为入站和出站连接计数.
		MaxPendingPeers int `toml:",omitempty"`
	
		// DialRatio控制入站和dialed connections的比率。 示例：DialRatio为2允许1/2 of connections to be dialed。 将DialRatio设置为零会将其默认为3。
		DialRatio int `toml:",omitempty"`
	
		// 可用于禁用peer发现机制。 禁用对协议调试（手动拓扑）很有用。
		NoDiscovery bool
	
		// 指定是否应启动基于主题发现的新V5发现协议。
		DiscoveryV5 bool `toml:",omitempty"`
	
		// 此服务器的节点名称。 使用common.MakeName创建遵循现有约定的名称。
		Name string `toml:"-"`
	
		// 用于与网络的其余部分建立连接。
		BootstrapNodes []*discover.Node
	
		// 用于使用V5发现协议与网络的其余部分建立连接。
		BootstrapNodesV5 []*discv5.Node `toml:",omitempty"`
	
		// 静态节点用作预配置的连接，始终在断开连接时进行维护和重新连接。
		StaticNodes []*discover.Node
	
		// 可信节点用作预先配置的连接，始终允许连接，甚至高于peer限制。
		TrustedNodes []*discover.Node
	
		// 限制特定的IP链接
		NetRestrict *netutil.Netlist `toml:",omitempty"`
	
		// 包含先前在网络中看到的活动节点的数据库的路径。
		NodeDatabase string `toml:",omitempty"`
	
		// 协议应包含服务器支持的协议。 为每个peer启动匹配协议。
		Protocols []Protocol `toml:"-"`
	
		//如果ListenAddr设置为非零地址，则服务器将侦听传入连接。
		//如果端口为零，操作系统将选择一个端口。 启动服务器时，将使用实际地址更新ListenAddr字段。
		ListenAddr string
	
		// 如果设置为非零值，则使用给定的NAT端口映射器使侦听端口可用于Internet。
		NAT nat.Interface `toml:",omitempty"`
	
		// 如果将Dialer设置为非零值，则使用给定的Dialer拨打出站peer连接。
		Dialer NodeDialer `toml:"-"`
	
		// 如果NoDial为true，则服务器不会拨打任何peer
		NoDial bool `toml:",omitempty"`
	
		//如果设置了EnableMsgEvents，则只要向对等方发送消息或从对等方接收消息，服务器就会发出PeerEvent
		EnableMsgEvents bool
	
		// 日志
		Logger log.Logger `toml:",omitempty"`
	}

关于BootstrapNodes，StaticNodes，TrustedNodes的区别-todo



Server 的结构体，管理所有节点的连接

	type Server struct {
		// 服务启动的时候不能被改变
		Config
	
		// Hooks for testing. These are useful because we can inhibit
		// the whole protocol stack.
		newTransport func(net.Conn) transport
		newPeerHook  func(*Peer)
	
		lock    sync.Mutex // 保护运行
		running bool
	
		nodedb       *enode.DB	//DB是节点数据库，存储先前看到的节点以及任何收集的有关它们的元数据以用于QoS目的。
		localnode    *enode.LocalNode //LocalNode生成本地节点的签名节点记录，即在当前进程中运行的节点
		ntab         discoverTable  //发现节点存储节点的表
		listener     net.Listener  //监听
		ourHandshake *protoHandshake
		lastLookup   time.Time  //最后一次查找的时间
		DiscV5       *discv5.Network
	
		// These are for Peers, PeerCount (and nothing else).
		peerOp     chan peerOpFunc
		peerOpDone chan struct{}
	
		quit          chan struct{}
		addstatic     chan *enode.Node
		removestatic  chan *enode.Node
		addtrusted    chan *enode.Node
		removetrusted chan *enode.Node
		posthandshake chan *conn
		addpeer       chan *conn
		delpeer       chan peerDrop
		loopWG        sync.WaitGroup // loop, listenLoop
		peerFeed      event.Feed
		log           log.Logger
	}

conn 结构 将两次握手暴露的信息封装起来

	type conn struct {
		fd net.Conn
		transport
		node  *enode.Node
		flags connFlag
		cont  chan error // 使用cont来向SetupConn发送信号错误。
		caps  []Cap      // 协议握手后有效
		name  string     // 协议握手后有效
	}

transport 接口

	type transport interface {
		// 两次握手
		doEncHandshake(prv *ecdsa.PrivateKey, dialDest *ecdsa.PublicKey) (*ecdsa.PublicKey, error)
		doProtoHandshake(our *protoHandshake) (*protoHandshake, error)
		// MsgReadWriter只能在加密握手完成后使用。 代码使用conn.id通过在加密握手后将其设置为非零值来跟踪此情况。
		MsgReadWriter
		// transports must provide Close because we use MsgPipe in some of
		// the tests. Closing the actual network connection doesn't do
		// anything in those tests because NsgPipe doesn't use it.
		close(err error)
	}


1.1 Start方法运行服务，在停止后不能使用服务。


	func (srv *Server) Start() (err error) {
		srv.lock.Lock()
		defer srv.lock.Unlock()
		if srv.running { //判断服务是否已经启动
			return errors.New("server already running")
		}
		srv.running = true
		srv.log = srv.Config.Logger
		if srv.log == nil {
			srv.log = log.New()
		}
		// p2p服务无法使用
		if srv.NoDial && srv.ListenAddr == "" {
			srv.log.Warn("P2P server will be useless, neither dialing nor listening")
		}
	
		// static fields
		if srv.PrivateKey == nil {
			return fmt.Errorf("Server.PrivateKey must be set to a non-nil key")
		}
		// rlpx是实际（非测试）连接使用的传输协议。 它用帧锁和读/写截止时间包装帧编码器。
		if srv.newTransport == nil {
			srv.newTransport = newRLPX
		}
		if srv.Dialer == nil {
			srv.Dialer = TCPDialer{&net.Dialer{Timeout: defaultDialTimeout}}
		}
		//一系列初始化
		srv.quit = make(chan struct{})
		srv.addpeer = make(chan *conn)
		srv.delpeer = make(chan peerDrop)
		srv.posthandshake = make(chan *conn)
		srv.addstatic = make(chan *enode.Node)
		srv.removestatic = make(chan *enode.Node)
		srv.addtrusted = make(chan *enode.Node)
		srv.removetrusted = make(chan *enode.Node)
		srv.peerOp = make(chan peerOpFunc)
		srv.peerOpDone = make(chan struct{})
	
		if err := srv.setupLocalNode(); err != nil {
			return err
		}
		if srv.ListenAddr != "" {
			if err := srv.setupListening(); err != nil {
				return err
			}
		}
		if err := srv.setupDiscovery(); err != nil {
			return err
		}
		//返回最大动态链接数，默认max/3
		dynPeers := srv.maxDialedConns()
		//这里调用了dial的初始化方法
		dialer := newDialState(srv.localnode.ID(), srv.StaticNodes, srv.BootstrapNodes, srv.ntab, dynPeers, srv.NetRestrict)
		srv.loopWG.Add(1)
		//核心逻辑run
		go srv.run(dialer)
		return nil
	}

1.1.1 setupLocalNode

	func (srv *Server) setupLocalNode() error {
		// Create the devp2p handshake.
		pubkey := crypto.FromECDSAPub(&srv.PrivateKey.PublicKey)
		srv.ourHandshake = &protoHandshake{Version: baseProtocolVersion, Name: srv.Name, ID: pubkey[1:]}
		for _, p := range srv.Protocols {
			srv.ourHandshake.Caps = append(srv.ourHandshake.Caps, p.cap())
		}
		sort.Sort(capsByNameAndVersion(srv.ourHandshake.Caps))
	
		// Create the local node.
		// 打开数据库
		db, err := enode.OpenDB(srv.Config.NodeDatabase)
		if err != nil {
			return err
		}
		srv.nodedb = db
		// 创建本地节点
		srv.localnode = enode.NewLocalNode(db, srv.PrivateKey)
		srv.localnode.SetFallbackIP(net.IP{127, 0, 0, 1})
		srv.localnode.Set(capsByNameAndVersion(srv.ourHandshake.Caps))
		// TODO: check conflicts
		//将节点的记录复制到本地？-todo
		for _, p := range srv.Protocols {
			for _, e := range p.Attributes {
				srv.localnode.Set(e)
			}
		}
		//选择NAT的方式？
		switch srv.NAT.(type) {
		case nil:
			// No NAT interface, do nothing.
		case nat.ExtIP:
			// ExtIP doesn't block, set the IP right away.
			ip, _ := srv.NAT.ExternalIP()
			srv.localnode.SetStaticIP(ip)
		default:
			// Ask the router about the IP. This takes a while and blocks startup,
			// do it in the background.
			srv.loopWG.Add(1)
			go func() {
				defer srv.loopWG.Done()
				if ip, err := srv.NAT.ExternalIP(); err == nil {
					srv.localnode.SetStaticIP(ip)
				}
			}()
		}
		return nil
	}

1.1.2 setupListening
配置监听

	func (srv *Server) setupListening() error {
		// 创建监听器
		listener, err := net.Listen("tcp", srv.ListenAddr)
		if err != nil {
			return err
		}
		laddr := listener.Addr().(*net.TCPAddr)
		srv.ListenAddr = laddr.String()
		srv.listener = listener                              
		srv.localnode.Set(enr.TCP(laddr.Port))
	
		srv.loopWG.Add(1)
		go srv.listenLoop()
	
		// 如果配置了NAT，则映射TCP侦听端口。
		// 判断是否是环回地址127.0.0.1
		if !laddr.IP.IsLoopback() && srv.NAT != nil {
			srv.loopWG.Add(1)
			go func() {
				nat.Map(srv.NAT, srv.quit, "tcp", laddr.Port, laddr.Port, "ethereum p2p")
				srv.loopWG.Done()
			}()
		}
		return nil
	}


1.1.2.1 listenLoop

listenLoop在自己的goroutine中运行并接受入站连接。

	func (srv *Server) listenLoop() {
		defer srv.loopWG.Done()
		srv.log.Debug("TCP listener up", "addr", srv.listener.Addr())
	
		tokens := defaultMaxPendingPeers
		if srv.MaxPendingPeers > 0 {
			tokens = srv.MaxPendingPeers
		}
		slots := make(chan struct{}, tokens)
		for i := 0; i < tokens; i++ {
			slots <- struct{}{}
		}
	
		for {
			// Wait for a handshake slot before accepting.
			<-slots
	
			var (
				fd  net.Conn
				err error
			)
			for {
				fd, err = srv.listener.Accept()
				if netutil.IsTemporaryError(err) {
					srv.log.Debug("Temporary read error", "err", err)
					continue
				} else if err != nil {
					srv.log.Debug("Read error", "err", err)
					return
				}
				break
			}
	
			// Reject connections that do not match NetRestrict.
			// 不包括在NetRestrict中的地址将会被拒绝
			if srv.NetRestrict != nil {
				if tcp, ok := fd.RemoteAddr().(*net.TCPAddr); ok && !srv.NetRestrict.Contains(tcp.IP) {
					srv.log.Debug("Rejected conn (not whitelisted in NetRestrict)", "addr", fd.RemoteAddr())
					fd.Close()
					slots <- struct{}{}
					continue
				}
			}
	
			var ip net.IP
			if tcp, ok := fd.RemoteAddr().(*net.TCPAddr); ok {
				ip = tcp.IP
			}
			fd = newMeteredConn(fd, true, ip)
			srv.log.Trace("Accepted connection", "addr", fd.RemoteAddr())
			go func() {
				srv.SetupConn(fd, inboundConn, nil)
				slots <- struct{}{}
			}()
		}
	}

1.1.2.1.1 SetupConn

SetupConn运行握手并尝试将连接添加为对等方。 当连接作为对等体添加或握手失败时返回。

	func (srv *Server) SetupConn(fd net.Conn, flags connFlag, dialDest *enode.Node) error {
		c := &conn{fd: fd, transport: srv.newTransport(fd), flags: flags, cont: make(chan error)}
		err := srv.setupConn(c, flags, dialDest)
		if err != nil {
			c.close(err)
			srv.log.Trace("Setting up connection failed", "addr", fd.RemoteAddr(), "err", err)
		}
		return err
	}


	func (srv *Server) setupConn(c *conn, flags connFlag, dialDest *enode.Node) error {
		// 防止剩余的待处理的conns进入握手。
		srv.lock.Lock()
		running := srv.running
		srv.lock.Unlock()
		if !running {
			return errServerStopped
		}
		// 如果正在dial，找出远程公钥。
		var dialPubkey *ecdsa.PublicKey
		//在server listenLoop调用中dialDest为nil
		if dialDest != nil {
			dialPubkey = new(ecdsa.PublicKey)
			if err := dialDest.Load((*enode.Secp256k1)(dialPubkey)); err != nil {
				return fmt.Errorf("dial destination doesn't have a secp256k1 public key")
			}
		}
		// Run the encryption handshake.第一次
		// 取得远程节点的公钥和自己的私钥进行加密握手
		remotePubkey, err := c.doEncHandshake(srv.PrivateKey, dialPubkey)
		if err != nil {
			srv.log.Trace("Failed RLPx handshake", "addr", c.fd.RemoteAddr(), "conn", c.flags, "err", err)
			return err
		}
		// 加密握手完毕后检测

		if dialDest != nil { // 发起方

			// For dialed connections, check that the remote public key matches.
			if dialPubkey.X.Cmp(remotePubkey.X) != 0 || dialPubkey.Y.Cmp(remotePubkey.Y) != 0 {
				return DiscUnexpectedIdentity
			}
			c.node = dialDest
		} else { //握手的接收方
			c.node = nodeFromConn(remotePubkey, c.fd)
		}
		if conn, ok := c.fd.(*meteredConn); ok {
			conn.handshakeDone(c.node.ID())
		}
		clog := srv.log.New("id", c.node.ID(), "addr", c.fd.RemoteAddr(), "conn", c.flags)
		err = srv.checkpoint(c, srv.posthandshake)
		if err != nil {
			clog.Trace("Rejected peer before protocol handshake", "err", err)
			return err
		}
		// Run the protocol handshake 第二次
		phs, err := c.doProtoHandshake(srv.ourHandshake)
		if err != nil {
			clog.Trace("Failed proto handshake", "err", err)
			return err
		}
		if id := c.node.ID(); !bytes.Equal(crypto.Keccak256(phs.ID), id[:]) {
			clog.Trace("Wrong devp2p handshake identity", "phsid", fmt.Sprintf("%x", phs.ID))
			return DiscUnexpectedIdentity
		}
		c.caps, c.name = phs.Caps, phs.Name
		//握手完毕,将新连接对象*p2p.conn压入server.addpeer
		err = srv.checkpoint(c, srv.addpeer)
		if err != nil {
			clog.Trace("Rejected peer", "err", err)
			return err
		}
		// If the checks completed successfully, runPeer has now been
		// launched by run.
		clog.Trace("connection set up", "inbound", dialDest == nil)
		return nil
	}

1.1.3 setupDiscovery

	func (srv *Server) setupDiscovery() error {
		if srv.NoDiscovery && !srv.DiscoveryV5 {
			return nil
		}
		//ResolveTCPAddr将addr作为TCP地址解析并返回。参数addr格式为"host:port"或"[ipv6-host%zone]:port"，解析得到网络名和端口名
		//解析返回一个UDP终端地址addr
		addr, err := net.ResolveUDPAddr("udp", srv.ListenAddr)
		if err != nil {
			return err
		}
		//ListenUDP创建一个接收目的地是本地地址addr的UDP数据包的网络连接。
		
		conn, err := net.ListenUDP("udp", addr)
		if err != nil {
			return err
		}
		// 本地网络地址
		realaddr := conn.LocalAddr().(*net.UDPAddr)
		srv.log.Debug("UDP listener up", "addr", realaddr)
		// 映射udp监听端口
		if srv.NAT != nil {
			if !realaddr.IP.IsLoopback() {
				go nat.Map(srv.NAT, srv.quit, "udp", realaddr.Port, realaddr.Port, "ethereum discovery")
			}
		}
		srv.localnode.SetFallbackUDP(realaddr.Port)
	
		// Discovery V4
		var unhandled chan discover.ReadPacket
		var sconn *sharedUDPConn
		if !srv.NoDiscovery {
			if srv.DiscoveryV5 {
				unhandled = make(chan discover.ReadPacket, 100)
				sconn = &sharedUDPConn{conn, unhandled}
			}
			cfg := discover.Config{
				PrivateKey:  srv.PrivateKey,
				NetRestrict: srv.NetRestrict,
				Bootnodes:   srv.BootstrapNodes,
				Unhandled:   unhandled,
			}
			//启动discover网络。 
			//返回一个新表，用于侦听laddr上的UDP数据包。
			ntab, err := discover.ListenUDP(conn, srv.localnode, cfg)
			if err != nil {
				return err
			}
			srv.ntab = ntab
		}
		// Discovery V5
		//这是新的节点发现协议。
		if srv.DiscoveryV5 {
			var ntab *discv5.Network
			var err error
			if sconn != nil {
				ntab, err = discv5.ListenUDP(srv.PrivateKey, sconn, "", srv.NetRestrict)
			} else {
				ntab, err = discv5.ListenUDP(srv.PrivateKey, conn, "", srv.NetRestrict)
			}
			if err != nil {
				return err
			}
			if err := ntab.SetFallbackNodes(srv.BootstrapNodesV5); err != nil {
				return err
			}
			srv.DiscV5 = ntab
		}
		return nil
	}

1.1.4 run 

核心逻辑

	func (srv *Server) run(dialstate dialer) {
		srv.log.Info("Started P2P networking", "self", srv.localnode.Node())
		defer srv.loopWG.Done()
		defer srv.nodedb.Close()
	
		var (
			peers        = make(map[enode.ID]*Peer)
			inboundCount = 0
			trusted      = make(map[enode.ID]bool, len(srv.TrustedNodes))
			taskdone     = make(chan task, maxActiveDialTasks) //16个最大任务量
			runningTasks []task  //正在执行
			queuedTasks  []task // 没有执行的
		)
		// 将受信任的节点放入map加快查找速度
		// 可信对等体在启动时加载或通过AddTrustedPeer RPC添加。
		for _, n := range srv.TrustedNodes {
			trusted[n.ID()] = true
		}
	
		// 这是一个将任务从正在运行的任务列表中删除的方法
		delTask := func(t task) {
			for i := range runningTasks {
				if runningTasks[i] == t {
					runningTasks = append(runningTasks[:i], runningTasks[i+1:]...)
					break
				}
			}
		}
		// 启动，直到满足最大活动任务数
		// 遍历为每个任务开启goroutine执行该任务
		startTasks := func(ts []task) (rest []task) {
			i := 0
			for ; len(runningTasks) < maxActiveDialTasks && i < len(ts); i++ {
				t := ts[i]
				srv.log.Trace("New dial task", "task", t)
				go func() { t.Do(srv); taskdone <- t }()
				runningTasks = append(runningTasks, t)
			}
			return ts[i:]
		}
		scheduleTasks := func() {
			// Start from queue first.
			// 这里startTasks将queuedTasks中超出最大动态链接任务的任务，加入到queuedTasks之前
			queuedTasks = append(queuedTasks[:0], startTasks(queuedTasks)...)
			// Query dialer for new tasks and start as many as possible now.
			if len(runningTasks) < maxActiveDialTasks {
				//调用newTasks来生成任务，并尝试用startTasks启动。并把暂时无法启动的放入queuedTasks队列
				nt := dialstate.newTasks(len(runningTasks)+len(queuedTasks), peers, time.Now())
				queuedTasks = append(queuedTasks, startTasks(nt)...)
			}
		}
	
	running:
		for {
			scheduleTasks()
	
			select {
			case <-srv.quit:
				// 服务被停止，运行清理逻辑
				break running
			case n := <-srv.addstatic:
				// AddPeer使用此通道添加到临时静态对等列表。 将它添加到dial，它将保持节点连接。
				srv.log.Trace("Adding static node", "node", n)
				dialstate.addStatic(n)
			case n := <-srv.removestatic:
				// RemovePeer使用此通道向对等方发送断开连接请求，并开始停止保持节点连接。
				srv.log.Trace("Removing static node", "node", n)
				dialstate.removeStatic(n)
				if p, ok := peers[n.ID()]; ok {
					p.Disconnect(DiscRequested)
				}
			case n := <-srv.addtrusted:
				// AddTrustedPeer 使用这个通道添加信任节点
				srv.log.Trace("Adding trusted node", "node", n)
				trusted[n.ID()] = true
				// 将已经连接的peer标记为信任
				if p, ok := peers[n.ID()]; ok {
					p.rw.set(trustedConn, true)
				}
			case n := <-srv.removetrusted:
				// RemoveTrustedPeer使用这个通道删除信任集合中的信任节点
				srv.log.Trace("Removing trusted node", "node", n)
				if _, ok := trusted[n.ID()]; ok {
					delete(trusted, n.ID())
				}
				// 取消信任标记
				if p, ok := peers[n.ID()]; ok {
					p.rw.set(trustedConn, false)
				}
			case op := <-srv.peerOp:
				// Peers and PeerCount 的操作.
				op(peers)
				srv.peerOpDone <- struct{}{}
			case t := <-taskdone:
				// 一个任务完成，通知dialstate，并将该任务从任务列表删除
				srv.log.Trace("Dial task done", "task", t)
				dialstate.taskDone(t, time.Now())
				delTask(t)
			case c := <-srv.posthandshake:
				// 连接已通过加密握手，因此远程身份已知（但尚未验证）。
				if trusted[c.node.ID()] {
					// 确保在检查MaxPeers之前设置了可信标志。
					c.flags |= trustedConn
				}
				// TODO: track in-progress inbound node IDs (pre-Peer) to avoid dialing them.
				select {
				case c.cont <- srv.encHandshakeChecks(peers, inboundCount, c):
				case <-srv.quit:
					break running
				}
			case c := <-srv.addpeer:
				// 此时连接通过了协议握手。 它的功能是已知的，并且验证了远程身份。
				err := srv.protoHandshakeChecks(peers, inboundCount, c)
				
				if err == nil {
					// 握手完成通过所有检查
					// 创建节点peer对象,传入所有子协议实现,自己实现的子协议就是在这里传入peer的,传入的所有协议通过matchProtocols函数格式化组织
					p := newPeer(c, srv.Protocols)
					//如果启用了消息事件，请将peerFeed传递给对等方
					if srv.EnableMsgEvents {
						p.events = &srv.peerFeed
					}
					name := truncateName(c.name)
					srv.log.Debug("Adding p2p peer", "name", name, "addr", c.fd.RemoteAddr(), "peers", len(peers)+1)
					//runPeer为每个对等体运行自己的goroutine。 它一直等到Peer逻辑返回并删除对等体。
					go srv.runPeer(p)
					peers[c.node.ID()] = p
					if p.Inbound() {
						inboundCount++
					}
				}
				// The dialer logic relies on the assumption that
				// dial tasks complete after the peer has been added or
				// discarded. Unblock the task last.
				select {
				case c.cont <- err:
				case <-srv.quit:
					break running
				}
			case pd := <-srv.delpeer:
				// 断开与某一个peer的链接
				d := common.PrettyDuration(mclock.Now() - pd.created)
				pd.log.Debug("Removing p2p peer", "duration", d, "peers", len(peers)-1, "req", pd.requested, "err", pd.err)
				delete(peers, pd.ID())
				if pd.Inbound() {
					inboundCount--
				}
			}
		}
	
		srv.log.Trace("P2P networking is spinning down")
	
		// Terminate discovery. If there is a running lookup it will terminate soon.
		if srv.ntab != nil {
			srv.ntab.Close()
		}
		if srv.DiscV5 != nil {
			srv.DiscV5.Close()
		}
		// 断开所有链接
		for _, p := range peers {
			p.Disconnect(DiscQuitting)
		}
		// 等待peer关闭。 未处理挂起的连接和任务，并且很快就会终止 - 因为srv.quit已关闭。
		for len(peers) > 0 {
			p := <-srv.delpeer
			p.log.Trace("<-delpeer (spindown)", "remainingTasks", len(runningTasks))
			delete(peers, p.ID())
		}
	}



1.1.4.1 encHandshakeChecks

检查encHandshake

	func (srv *Server) encHandshakeChecks(peers map[enode.ID]*Peer, inboundCount int, c *conn) error {
		switch {
		//不是新人连接或者静态链接并且peer数大于最大链接，就会返回动态链接过多的错误
		case !c.is(trustedConn|staticDialedConn) && len(peers) >= srv.MaxPeers:
			return DiscTooManyPeers
		//太多入站数
		case !c.is(trustedConn) && c.is(inboundConn) && inboundCount >= srv.maxInboundConns():
			return DiscTooManyPeers
		//链接已经存在
		case peers[c.node.ID()] != nil:
			return DiscAlreadyConnected
		//不能是自己本身
		case c.node.ID() == srv.localnode.ID():
			return DiscSelf
		default:
			return nil
		}
	}

1.1.4.2 protoHandshakeChecks

	func (srv *Server) protoHandshakeChecks(peers map[enode.ID]*Peer, inboundCount int, c *conn) error {
		// 删除没有匹配协议的连接。
		if len(srv.Protocols) > 0 && countMatchingProtocols(srv.Protocols, c.caps) == 0 {
			return DiscUselessPeer
		}
		// 重复加密握手检查，因为peer设置可能在握手之间发生了变化。
		return srv.encHandshakeChecks(peers, inboundCount, c)
	}



1.1.4.3 runPeer

runPeer为每个对等体运行自己的goroutine。 它一直等到Peer逻辑返回并删除对等体。

	func (srv *Server) runPeer(p *Peer) {
		if srv.newPeerHook != nil {
			srv.newPeerHook(p)
		}
	
		// broadcast peer add
		srv.peerFeed.Send(&PeerEvent{
			Type: PeerEventTypeAdd,
			Peer: p.ID(),
		})
	
		// run the protocol
		remoteRequested, err := p.run()
	
		// broadcast peer drop
		srv.peerFeed.Send(&PeerEvent{
			Type:  PeerEventTypeDrop,
			Peer:  p.ID(),
			Error: err.Error(),
		})
	
		// Note: run waits for existing peers to be sent on srv.delpeer
		// before returning, so this send should not select on srv.quit.
		srv.delpeer <- peerDrop{p, err, remoteRequested}
	}

## 总结 ##

----------
server对象主要完成的工作把之前介绍的所有组件组合在一起。 使用rlpx.go来处理加密链路。 使用discover来处理节点发现和查找。 使用dial来生成和连接需要连接的节点。 使用peer对象来处理每个连接。

server启动了一个listenLoop来监听和接收新的连接。 启动一个run的goroutine来调用dialstate生成新的dial任务并进行连接。 goroutine之间使用channel来进行通讯和配合。