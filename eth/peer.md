# peer #

----------
eth peer 的结构体 

	type peer struct {
		id string
	
		*p2p.Peer	// p2p的peer
		rw p2p.MsgReadWriter	// 消息读写
	
		version  int         // Protocol version negotiated
		forkDrop *time.Timer // Timed connection dropper if forks aren't validated in time
	
		head common.Hash		//
		td   *big.Int		//难度
		lock sync.RWMutex
	
		//这个peer已知的事务哈希集
		knownTxs    mapset.Set                // Set of transaction hashes known to be known by this peer
		//这个peer已知的块哈希
		knownBlocks mapset.Set                // Set of block hashes known to be known by this peer
		// 要向同级广播的事务队列
		queuedTxs   chan []*types.Transaction // Queue of transactions to broadcast to the peer
		// 要向peer广播的块的队列
		queuedProps chan *propEvent           // Queue of blocks to broadcast to the peer
		// 要像peer声明的块的队列
		queuedAnns  chan *types.Block         // Queue of blocks to announce to the peer
		// 关闭通道
		term        chan struct{}             // Termination channel to stop the broadcaster
	}


握手执行eth协议握手，协商版本号，网络ID，难度 ，头部和创建块。


AsyncSendTransactions将传播到远程对等方的事务列表排队。 如果对等方的广播队列已满，则会以静默方式删除该事件。
	
	func (p *peer) AsyncSendTransactions(txs []*types.Transaction) {
		select {
		case p.queuedTxs <- txs:
			for _, tx := range txs {
				p.knownTxs.Add(tx.Hash())// 添加到本节点的已知事务上
			}
		default:
			p.Log().Debug("Dropping transaction propagation", "count", len(txs))
		}
	}


