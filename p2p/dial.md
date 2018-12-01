# dial #

----------
	
Tcp节点挑选和连接的处理主要在dial

p2p里面主要负责建立链接的部分工作。 比如发现建立链接的节点。 与节点建立链接。 通过discover来查找指定节点的地址等功能。

	//计划拨号和发现查找。 它有机会在Server.run的主循环的每次迭代中计算新任务。
	type dialstate struct {
		maxDynDials int  //最大的动态节点链接数量
		ntab        discoverTable  //节点发现
		netrestrict *netutil.Netlist
	
		lookupRunning bool
		dialing       map[discover.NodeID]connFlag   //正在链接的节点
		lookupBuf     []*discover.Node // 当前已经寻找到的节点
		randomNodes   []*discover.Node // discoverTable随机查询的节点
		static        map[discover.NodeID]*dialTask  //静态的节点。
		hist          *dialHistory   //dial的历史记录
	
		start     time.Time        // time when the dialer was first used
		bootnodes []*discover.Node // 刚开始的建立的dial
	}

dialTask

为每个dial的节点生成dialTask。 任务运行时无法访问其字段。


	type dialTask struct {
		flags        connFlag
		dest         *discover.Node
		lastResolved time.Time
		resolveDelay time.Duration
	}

newDialState
初始化

	func newDialState(static []*discover.Node, bootnodes []*discover.Node, ntab discoverTable, maxdyn int, netrestrict *netutil.Netlist) *dialstate {
		s := &dialstate{
			maxDynDials: maxdyn,
			ntab:        ntab,
			netrestrict: netrestrict,
			static:      make(map[discover.NodeID]*dialTask),
			dialing:     make(map[discover.NodeID]connFlag),
			bootnodes:   make([]*discover.Node, len(bootnodes)),
			randomNodes: make([]*discover.Node, maxdyn/2),
			hist:        new(dialHistory),
		}
		copy(s.bootnodes, bootnodes)
		//添加静态节点
		//这里静态节点和bootnodes有一些不同，我理解为静态节点是自己可以设定重连时选择与之链接，bootnodes是为了与更多的网络中其他的节点链接而设置的
		for _, n := range static {
			s.addStatic(n)
		}
		return s
	}

addStatic


	func (s *dialstate) addStatic(n *discover.Node) {
		// 这会覆盖任务而不是更新现有条目，从而使用户有机会强制执行解析操作。
		s.static[n.ID] = &dialTask{flags: staticDialedConn, dest: n}
	}

1.1 newTasks

主要逻辑

	func (s *dialstate) newTasks(nRunning int, peers map[discover.NodeID]*Peer, now time.Time) []task {
		if s.start.IsZero() { //如果开始时间为0，赋值开始时间为当前时间
			s.start = now
		}
	
		var newtasks []task
		addDial := func(flag connFlag, n *discover.Node) bool {
			//检查是否需要建立链接

			if err := s.checkDial(n, peers); err != nil {
				log.Trace("Skipping dial candidate", "id", n.ID, "addr", &net.TCPAddr{IP: n.IP, Port: int(n.TCP)}, "err", err)
				return false
			}
			s.dialing[n.ID] = flag
			//然后设置状态，最后把节点增加到newtasks队列里面。
			newtasks = append(newtasks, &dialTask{flags: flag, dest: n})
			return true
		}
	
		// 此时需要计算动态拨号的数量。
		//首先判断已经建立的连接的类型。如果是动态类型。那么需要建立动态链接数量减少。
		needDynDials := s.maxDynDials
		for _, p := range peers {
			if p.rw.is(dynDialedConn) { //不太懂--todo
				needDynDials--
			}
		}
		//然后再判断正在建立的链接。如果是动态类型。那么需要建立动态链接数量减少。
		for _, flag := range s.dialing {
			if flag&dynDialedConn != 0 {
				needDynDials--
			}
		}
	
		//在每次调用时历史记录到期。
		s.hist.expire(now)
	
		// 如果未连接，则为静态节点创建拨号。
		for id, t := range s.static {
			err := s.checkDial(t.dest, peers)
			switch err {
			case errNotWhitelisted, errSelf:
				log.Warn("Removing static dial candidate", "id", t.dest.ID, "addr", &net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)}, "err", err)
				delete(s.static, t.dest.ID)
			case nil:
				s.dialing[id] = t.flags
				newtasks = append(newtasks, t)
			}
		}
		// 如果我们没有任何对等方，请尝试链接随机bootnode。 这种情况对于testnet（和专用网络）非常有用，在这种情况下，发现表可能充满了大多数不良对等体，因此很难找到好的对等体。
		//如果当前还没有任何链接。 而且20秒(fallbackInterval)内没有创建任何链接。 那么就使用bootnode创建链接。
		if len(peers) == 0 && len(s.bootnodes) > 0 && needDynDials > 0 && now.Sub(s.start) > fallbackInterval {
			//与首位bootnodes创建链接并将它放到最后
			bootnode := s.bootnodes[0]
			s.bootnodes = append(s.bootnodes[:0], s.bootnodes[1:]...)
			s.bootnodes = append(s.bootnodes, bootnode)
	
			if addDial(dynDialedConn, bootnode) {
				needDynDials--
			}

		// 使用表中的随机节点获取一半必要的动态链接
		randomCandidates := needDynDials / 2
		if randomCandidates > 0 {
			n := s.ntab.ReadRandomNodes(s.randomNodes)
			for i := 0; i < randomCandidates && i < n; i++ {
				if addDial(dynDialedConn, s.randomNodes[i]) {
					needDynDials--
				}
			}
		}
		// Create dynamic dials from random lookup results, removing tried
		// items from the result buffer.
		//从随机查找的结果中创建动态链接，并将他们从果缓存中移除
		i := 0
		for ; i < len(s.lookupBuf) && needDynDials > 0; i++ {
			if addDial(dynDialedConn, s.lookupBuf[i]) {
				needDynDials--
			}
		}
		//将刚才加入链接的删除
		s.lookupBuf = s.lookupBuf[:copy(s.lookupBuf, s.lookupBuf[i:])]
		//如果就算这样也不能创建足够动态链接。 那么创建一个discoverTask用来再网络上查找其他的节点。放入lookupBuf
		if len(s.lookupBuf) < needDynDials && !s.lookupRunning {
			s.lookupRunning = true
			//任务中添加发现节点任务
			newtasks = append(newtasks, &discoverTask{})
		}
	
		// 如果当前没有任何任务需要做，那么创建一个睡眠的任务返回。
		
		if nRunning == 0 && len(newtasks) == 0 && s.hist.Len() > 0 {
			t := &waitExpireTask{s.hist.min().exp.Sub(now)}
			newtasks = append(newtasks, t)
		}
		return newtasks
	}

task

接口

	type task interface {
		Do(*Server)
	}


1.2 dialTask的Do方法

主要的链接方法

	func (t *dialTask) Do(srv *Server) {
		//没有IP地址的节点返回true
		if t.dest.Incomplete() {
			if !t.resolve(srv) {
				return
			}
		}
		err := t.dial(srv, t.dest)
		if err != nil {
			log.Trace("Dial error", "task", t, "err", err)
			// Try resolving the ID of static nodes if dialing failed.
			if _, ok := err.(*dialError); ok && t.flags&staticDialedConn != 0 {
				if t.resolve(srv) {
					t.dial(srv, t.dest)
				}
			}
		}
	}

1.1.1 checkDial

 用来检查任务是否需要创建链接。
	
	func (s *dialstate) checkDial(n *discover.Node, peers map[discover.NodeID]*Peer) error {
		_, dialing := s.dialing[n.ID]
		switch {
		case dialing: //正在进行连接
			return errAlreadyDialing
		case peers[n.ID] != nil:	//已经有连接
			return errAlreadyConnected
		case s.ntab != nil && n.ID == s.ntab.Self().ID:  //不能与自己建立连接
			return errSelf
		case s.netrestrict != nil && !s.netrestrict.Contains(n.IP)://网络限制。 对方的IP地址不在白名单里面。
			return errNotWhitelisted
		case s.hist.contains(n.ID): // 这个ID曾经链接过。
			return errRecentlyDialed
		}
		return nil
	}


resolve

再table中的方法是按节点ID返回查找到的节点，或者

1.2.1 resolve操作受到退避限制，以避免对不存在的节点的无用查询充斥discovery网络。找到节点时，退避延迟将重置。
也就是说避免长时间的查找

	func (t *dialTask) resolve(srv *Server) bool {
		if srv.ntab == nil {
			log.Debug("Can't resolve node", "id", t.dest.ID, "err", "discovery is disabled")
			return false
		}
		//初始化resolveDelay为60秒
		if t.resolveDelay == 0 {
			t.resolveDelay = initialResolveDelay
		}
		//如果没到期，也就是距离上次查找还没到60秒，返回false
		if time.Since(t.lastResolved) < t.resolveDelay {
			return false
		}
		resolved := srv.ntab.Resolve(t.dest.ID)
		t.lastResolved = time.Now()
		if resolved == nil {
			//如果不存在则resolveDelay翻倍
			t.resolveDelay *= 2
			//如果resolveDelay最终大于maxResolveDelay一个小时，按一个小时算
			if t.resolveDelay > maxResolveDelay {
				t.resolveDelay = maxResolveDelay
			}
			log.Debug("Resolving node failed", "id", t.dest.ID, "newdelay", t.resolveDelay)
			return false
		}

		// 找到的话就重置60秒
		
		t.resolveDelay = initialResolveDelay
		t.dest = resolved
		log.Debug("Resolved node", "id", t.dest.ID, "addr", &net.TCPAddr{IP: t.dest.IP, Port: int(t.dest.TCP)})
		return true
	}

1.2.2 dial

执行实际的连接过程

	func (t *dialTask) dial(srv *Server, dest *discover.Node) error {
		fd, err := srv.Dialer.Dial(dest)
		if err != nil {
			return &dialError{err}
		}
		mfd := newMeteredConn(fd, false)
		//主要通过srv.SetupConn方法来完成
		return srv.SetupConn(mfd, t.flags, dest)
	}

Dial

与节点创建TCP链接

	func (t TCPDialer) Dial(dest *discover.Node) (net.Conn, error) {
		addr := &net.TCPAddr{IP: dest.IP, Port: int(dest.TCP)}
		return t.Dialer.Dial("tcp", addr.String())
	}


discoverTask的Do方法

每隔一定时间进行查找节点

	func (t *discoverTask) Do(srv *Server) {
		//每当需要动态链接时，newTasks都会生成查找任务。 查找需要花一些时间，否则事件循环旋转太快。
		// event loop spins too fast.
		//有一个4秒钟的查找间隔
		next := srv.lastLookup.Add(lookupInterval)
		if now := time.Now(); now.Before(next) {
			//等待间隔时间
			time.Sleep(next.Sub(now))
		}
		//更新
		srv.lastLookup = time.Now()
		var target discover.NodeID
		rand.Read(target[:])
		//进行随机查找
		t.results = srv.ntab.Lookup(target)
	}

waitExpireTask的Do方法

等待一定时间

func (t waitExpireTask) Do(*Server) {
	time.Sleep(t.Duration)
}


## 总结 ##

dial主要是负责建立节点之间的TCP链接。 主要逻辑在1.1 中，由server中调用。 这里addDial主要负责将合法的链接任务加入任务列表，
计算剩余的动态链接数。 这里的任务是一个接口，实现方法为Do方法，在dial中主要有三种任务，主要的dial任务为节点创建tcp链接，discover任务发现节点，等待任务。在1.1中分别针对不同的逻辑在任务i而表中添加相应的任务执行。