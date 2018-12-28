# handler #

----------
ProtocolManager 是以太坊协议主要的管理者
	
	type ProtocolManager struct {
		networkID uint64	// 网络ID判断是否是测试网
	
		// 快速同步
		fastSync  uint32 // Flag whether fast sync is enabled (gets disabled if we already have blocks)
		// 是否同步
		acceptTxs uint32 // Flag whether we're considered synchronised (enables transaction processing)
	
		txpool      txPool	//事务池
		blockchain  *core.BlockChain	//区块链
		chainconfig *params.ChainConfig	//链设置
		maxPeers    int	//最大节点设置
	
		downloader *downloader.Downloader	// 下载
		fetcher    *fetcher.Fetcher
		peers      *peerSet	//节点集
	
		SubProtocols []p2p.Protocol	//子协议
	
		eventMux      *event.TypeMux
		txsCh         chan core.NewTxsEvent	//当一批事务进入事务池时，将发布NewTxsEvent。
		txsSub        event.Subscription	//事务消息订阅
		minedBlockSub *event.TypeMuxSubscription //
	
		// channels for fetcher, syncer, txsyncLoop
		newPeerCh   chan *peer
		txsyncCh    chan *txsync
		quitSync    chan struct{}
		noMorePeers chan struct{}
	
		// wait group is used for graceful shutdowns during downloading
		// and processing
		wg sync.WaitGroup
	}

NewProtocolManager返回一个新的以太坊子协议管理器。 以太坊子协议管理能够与以太坊网络连接的对等体。


	func NewProtocolManager(config *params.ChainConfig, mode downloader.SyncMode, networkID uint64, mux *event.TypeMux, txpool txPool, engine consensus.Engine, blockchain *core.BlockChain, chaindb ethdb.Database) (*ProtocolManager, error) {
		// Create the protocol manager with the base fields
		// 初始化ProtocolManager
		manager := &ProtocolManager{
			networkID:   networkID,
			eventMux:    mux,
			txpool:      txpool,
			blockchain:  blockchain,
			chainconfig: config,
			peers:       newPeerSet(),
			newPeerCh:   make(chan *peer),
			noMorePeers: make(chan struct{}),
			txsyncCh:    make(chan *txsync),
			quitSync:    make(chan struct{}),
		}
		// Figure out whether to allow fast sync or not
		// 弄清楚是否允许快速同步
		if mode == downloader.FastSync && blockchain.CurrentBlock().NumberU64() > 0 {
			log.Warn("Blockchain not empty, fast sync disabled")
			mode = downloader.FullSync
		}
		if mode == downloader.FastSync {
			manager.fastSync = uint32(1)
		}
		// Initiate a sub-protocol for every implemented version we can handle
		// 初始化子协议
		manager.SubProtocols = make([]p2p.Protocol, 0, len(ProtocolVersions))
		for i, version := range ProtocolVersions {
			// Skip protocol version if incompatible with the mode of operation
			// 如果与操作模式不兼容，请跳过协议版本
			if mode == downloader.FastSync && version < eth63 {
				continue
			}
			// Compatible; initialise the sub-protocol
			// 初始化子协议
			version := version // Closure for the run
			manager.SubProtocols = append(manager.SubProtocols, p2p.Protocol{
				Name:    ProtocolName, //eth
				Version: version,
				Length:  ProtocolLengths[i],//ProtocolLengths是对应于不同协议版本的已实现消息的数量。
				Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
					peer := manager.newPeer(int(version), p, rw)
					select {
					case manager.newPeerCh <- peer:
						manager.wg.Add(1)
						defer manager.wg.Done()
						// 管理peer
						return manager.handle(peer)
					case <-manager.quitSync:
						return p2p.DiscQuitting
					}
				},
				NodeInfo: func() interface{} {
					return manager.NodeInfo()
				},
				PeerInfo: func(id enode.ID) interface{} {
					if p := manager.peers.Peer(fmt.Sprintf("%x", id[:8])); p != nil {
						return p.Info()
					}
					return nil
				},
			})
		}
		if len(manager.SubProtocols) == 0 {
			return nil, errIncompatibleConfig
		}
		// Construct the different synchronisation mechanisms
		// 构建不同的同步机制
		manager.downloader = downloader.New(mode, chaindb, manager.eventMux, blockchain, nil, manager.removePeer)
	
		// 验证者
		validator := func(header *types.Header) error {
			return engine.VerifyHeader(blockchain, header, true)
		}
		heighter := func() uint64 {
			return blockchain.CurrentBlock().NumberU64()
		}
		// 插入块
		inserter := func(blocks types.Blocks) (int, error) {
			// If fast sync is running, deny importing weird blocks
			// 如果正在运行快速同步，则拒绝导入奇怪的块
			if atomic.LoadUint32(&manager.fastSync) == 1 {
				log.Warn("Discarded bad propagated block", "number", blocks[0].Number(), "hash", blocks[0].Hash())
				return 0, nil
			}
			atomic.StoreUint32(&manager.acceptTxs, 1) // Mark initial sync done on any fetcher import
			return manager.blockchain.InsertChain(blocks)
		}
		// 块fetcher创建
		manager.fetcher = fetcher.New(blockchain.GetBlockByHash, validator, manager.BroadcastBlock, heighter, inserter, manager.removePeer)
	
		return manager, nil
	}

当p2p的server启动的时候，会主动的找节点去连接，或者被其他的节点连接。 连接的过程是首先进行加密信道的握手，然后进行协议的握手。 最后为每个协议启动goroutine 执行Run方法来把控制交给最终的协议。 这个run方法首先创建了一个peer对象，然后调用了handle方法来处理这个peer

			Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
				peer := manager.newPeer(int(version), p, rw)
				select {
				case manager.newPeerCh <- peer:
					manager.wg.Add(1)
					defer manager.wg.Done()
					// 管理peer
					return manager.handle(peer)
				case <-manager.quitSync:
					return p2p.DiscQuitting
				}
			},



handle是为了管理eth peer的生命周期而调用的回调。 当此函数终止时，对等体将断开连接。



Start这个方法里面启动了大量的goroutine用来处理各种事务，可以推测，这个类应该是以太坊服务的主要实现类。

	func (pm *ProtocolManager) Start(maxPeers int) {
		pm.maxPeers = maxPeers
	
		// broadcast transactions
		// 广播交易的通道。 txCh会作为txpool的TxPreEvent订阅通道。txpool有了这种消息会通知给这个txCh。 广播交易的goroutine会把这个消息广播出去。
		pm.txsCh = make(chan core.NewTxsEvent, txChanSize)
		// 订阅的回执
		pm.txsSub = pm.txpool.SubscribeNewTxsEvent(pm.txsCh)
		// 广播新出现的交易对象。
		// txBroadcastLoop()会在txCh通道的收端持续等待，一旦接收到有关新交易的事件，会立即调用BroadcastTx()函数广播给那些尚无该交易对象的相邻个体。
		go pm.txBroadcastLoop()
	
		// broadcast mined blocks
	
		pm.minedBlockSub = pm.eventMux.Subscribe(core.NewMinedBlockEvent{})
		// 广播新挖掘出的区块。minedBroadcastLoop()持续等待本个体的新挖掘出区块事件，然后立即广播给需要的相邻个体。
		go pm.minedBroadcastLoop()
	
		// start sync handlers
		// 定时与相邻个体进行区块全链的强制同步。
		go pm.syncer()
		// 将新出现的交易对象均匀的同步给相邻个体。
		go pm.txsyncLoop()
	}

广播交易

	func (pm *ProtocolManager) txBroadcastLoop() {
		for {
			select {
			case event := <-pm.txsCh://收到事务消息
				pm.BroadcastTxs(event.Txs)
	
			// Err() channel will be closed when unsubscribing.
			case <-pm.txsSub.Err():
				return
			}
		}
	}

BroadcastTxs会将一批事务传播给所有已知具有给定事务的对等体。

	func (pm *ProtocolManager) BroadcastTxs(txs types.Transactions) {
		var txset = make(map[*peer]types.Transactions)
	
		// Broadcast transactions to a batch of peers not knowing about it
		// 向一批不知道它的peer广播交易
		for _, tx := range txs {
			peers := pm.peers.PeersWithoutTx(tx.Hash())
			for _, peer := range peers {
				txset[peer] = append(txset[peer], tx)
			}
			log.Trace("Broadcast transaction", "hash", tx.Hash(), "recipients", len(peers))
		}
		// FIXME include this again: peers = peers[:int(math.Sqrt(float64(len(peers))))]
		for peer, txs := range txset {
			// 将事务发送给peer的事务等待队列
			peer.AsyncSendTransactions(txs)
		}
	}

minedBroadcastLoop 块广播
	
	func (pm *ProtocolManager) minedBroadcastLoop() {
		// automatically stops if unsubscribe
		// 如果取消订阅则自动停止
		for obj := range pm.minedBlockSub.Chan() {
			if ev, ok := obj.Data.(core.NewMinedBlockEvent); ok {
				// 第一次广播整个块给部分peer
				pm.BroadcastBlock(ev.Block, true)  // First propagate block to peers
				// 第二次广播块的哈希
				pm.BroadcastBlock(ev.Block, false) // Only then announce to the rest
			}
		}
	}


BroadcastBlock会将一个块传播到peers的一个子集，或者只宣告它的可用性（取决于所请求的）。


	func (pm *ProtocolManager) BroadcastBlock(block *types.Block, propagate bool) {
		hash := block.Hash()
		// 没这个块的peer
		peers := pm.peers.PeersWithoutBlock(hash)
	
		// If propagation is requested, send to a subset of the peer
		// 如果请求传播，则发送到对等体的子集
		// 第一次为true
		if propagate {
			// Calculate the TD of the block (it's not imported yet, so block.Td is not valid)
			// 计算块的TD（它尚未导入，因此block.Td无效）
			// 也就是说在块传播时还没设定该块的难度
			var td *big.Int
			// 取得父块
			if parent := pm.blockchain.GetBlock(block.ParentHash(), block.NumberU64()-1); parent != nil {
				// 计算难度
				td = new(big.Int).Add(block.Difficulty(), pm.blockchain.GetTd(block.ParentHash(), block.NumberU64()-1))
			} else {
				log.Error("Propagating dangling block", "number", block.Number(), "hash", hash)
				return
			}
			// Send the block to a subset of our peers
			// 发送该块到部分peer
			transferLen := int(math.Sqrt(float64(len(peers))))
			if transferLen < minBroadcastPeers { //最少向4个peer发送
				transferLen = minBroadcastPeers
			}
			if transferLen > len(peers) { //如果建立连接的peer少于4个那么就像所有peer发送
				transferLen = len(peers)
			}
			transfer := peers[:transferLen]
			for _, peer := range transfer {
				peer.AsyncSendNewBlock(block, td) //发送这个块
			}
			log.Trace("Propagated block", "hash", hash, "recipients", len(transfer), "duration", common.PrettyDuration(time.Since(block.ReceivedAt)))
			return
		}
		// Otherwise if the block is indeed in out own chain, announce it
		// 否则，如果该块确实在自己的链中，则对外声明它
		if pm.blockchain.HasBlock(hash, block.NumberU64()) {
			for _, peer := range peers { //向所有的peers声明
				peer.AsyncSendNewBlockHash(block)
			}
			log.Trace("Announced block", "hash", hash, "recipients", len(peers), "duration", common.PrettyDuration(time.Since(block.ReceivedAt)))
		}
	}