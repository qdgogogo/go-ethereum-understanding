# tx_pool #

----------
TxPool 包含了当前已知的交易， 当前网络接收到交易，或者本地提交的交易会加入到TxPool。 当他们已经被添加到区块链的时候被移除。

TxPool分为可执行的交易(可以应用到当前的状态)和未来的交易。 交易在这两种状态之间转换，

	type TxPool struct {
		config       TxPoolConfig
		chainconfig  *params.ChainConfig
		chain        blockChain
		gasPrice     *big.Int	//最低的GasPrice限制
		txFeed       event.Feed //通过txFeed来订阅TxPool的消息
		scope        event.SubscriptionScope
		chainHeadCh  chan ChainHeadEvent	// 订阅了区块头的消息，当有了新的区块头生成的时候会在这里收到通知
		chainHeadSub event.Subscription// 区块头消息的订阅器。
		signer       types.Signer	// 封装了事务签名处理。
		mu           sync.RWMutex
	
		currentState  *state.StateDB      // Current state in the blockchain head
		pendingState  *state.ManagedState // Pending state tracking virtual nonces
		currentMaxGas uint64              // Current gas limit for transaction caps
	
		//本地交易免除驱逐规则
		locals  *accountSet // Set of local transaction to exempt from eviction rules
		// 本地交易会写入磁盘
		journal *txJournal  // Journal of local transaction to back up to disk
	
		//所有当前可以处理的交易
		pending map[common.Address]*txList   // All currently processable transactions
		//当前还不能处理的交易
		queue   map[common.Address]*txList   // Queued but non-processable transactions
		// 每一个已知账号的最后一次心跳信息的时间
		beats   map[common.Address]time.Time // Last heartbeat from each known account
		// 可以查找到所有交易
		all     *txLookup                    // All transactions to allow lookups
		//按照价格排序的交易
		priced  *txPricedList                // All transactions sorted by price
	
		wg sync.WaitGroup // for shutdown sync
	
		// 是否是homestead版本
		homestead bool
	}


NewTxPool创建一个新的事务池，用于从网络收集，排序和过滤入站事务。

	func NewTxPool(config TxPoolConfig, chainconfig *params.ChainConfig, chain blockChain) *TxPool {
		// Sanitize the input to ensure no vulnerable gas prices are set
		// 消除输入以确保没有设定脆弱的天然气价格
		config = (&config).sanitize()
	
		// Create the transaction pool with its initial settings
		// 使用其初始设置创建事务池
		pool := &TxPool{
			config:      config,
			chainconfig: chainconfig,
			chain:       chain,
			signer:      types.NewEIP155Signer(chainconfig.ChainID),
			pending:     make(map[common.Address]*txList),
			queue:       make(map[common.Address]*txList),
			beats:       make(map[common.Address]time.Time),
			all:         newTxLookup(),
			chainHeadCh: make(chan ChainHeadEvent, chainHeadChanSize),
			gasPrice:    new(big.Int).SetUint64(config.PriceLimit),
		}
		pool.locals = newAccountSet(pool.signer)
		for _, addr := range config.Locals {
			log.Info("Setting new local account", "address", addr)
			pool.locals.add(addr)
		}
		pool.priced = newTxPricedList(pool.all)
		pool.reset(nil, chain.CurrentBlock().Header())
	
		// If local transactions and journaling is enabled, load from disk
		// 如果启用了本地事务和日记功能，请从磁盘加载
		if !config.NoLocals && config.Journal != "" {
			pool.journal = newTxJournal(config.Journal)
	
			if err := pool.journal.load(pool.AddLocals); err != nil {
				log.Warn("Failed to load transaction journal", "err", err)
			}
			if err := pool.journal.rotate(pool.local()); err != nil {
				log.Warn("Failed to rotate transaction journal", "err", err)
			}
		}
		// Subscribe events from blockchain
		// 订阅区块链消息
		pool.chainHeadSub = pool.chain.SubscribeChainHeadEvent(pool.chainHeadCh)
	
		// Start the event loop and return
		// 开启事务循环
		pool.wg.Add(1)
		go pool.loop()
	
		return pool
	}



sanitize检查提供的用户配置并更改任何不合理或不可行的内容。
	
	func (config *TxPoolConfig) sanitize() TxPoolConfig {
		conf := *config
		if conf.Rejournal < time.Second {
			log.Warn("Sanitizing invalid txpool journal time", "provided", conf.Rejournal, "updated", time.Second)
			conf.Rejournal = time.Second
		}
		if conf.PriceLimit < 1 {
			log.Warn("Sanitizing invalid txpool price limit", "provided", conf.PriceLimit, "updated", DefaultTxPoolConfig.PriceLimit)
			conf.PriceLimit = DefaultTxPoolConfig.PriceLimit
		}
		if conf.PriceBump < 1 {
			log.Warn("Sanitizing invalid txpool price bump", "provided", conf.PriceBump, "updated", DefaultTxPoolConfig.PriceBump)
			conf.PriceBump = DefaultTxPoolConfig.PriceBump
		}
		return conf
	}



loop是事务池的主要事件循环，等待并响应外部区块链事件以及各种报告和事务驱逐事件。

	func (pool *TxPool) loop() {
		defer pool.wg.Done()
	
		// Start the stats reporting and transaction eviction tickers
		// 启动统计报告和交易逐出代码
		var prevPending, prevQueued, prevStales int
		// 8秒钟的报告事务池状态间隔
		report := time.NewTicker(statsReportInterval)
		defer report.Stop()
	
		//1分钟的检查可撤销事务的间隔
		evict := time.NewTicker(evictionInterval)
		defer evict.Stop()
	
		//重新生成本地事务日志的时间间隔
		journal := time.NewTicker(pool.config.Rejournal)
		defer journal.Stop()
	
		// Track the previous head headers for transaction reorgs
		// 跟踪先前的头部标题以进行事务重组
		head := pool.chain.CurrentBlock()
	
		// Keep waiting for and reacting to the various events
	
		// 继续等待和响应各种事件
		for {
			select {
			// Handle ChainHeadEvent
			// 处理ChainHeadEvent
			case ev := <-pool.chainHeadCh:
				if ev.Block != nil {
					pool.mu.Lock()
					if pool.chainconfig.IsHomestead(ev.Block.Number()) {
						pool.homestead = true
					}
					pool.reset(head.Header(), ev.Block.Header())
					head = ev.Block
	
					pool.mu.Unlock()
				}
			// Be unsubscribed due to system stopped
			// 由于系统停止而取消订阅
			case <-pool.chainHeadSub.Err():
				return
	
			// Handle stats reporting ticks
			// 处理状态报告
			case <-report.C:
				pool.mu.RLock()
				pending, queued := pool.stats()
				stales := pool.priced.stales
				pool.mu.RUnlock()
	
				if pending != prevPending || queued != prevQueued || stales != prevStales {
					log.Debug("Transaction pool status report", "executable", pending, "queued", queued, "stales", stales)
					prevPending, prevQueued, prevStales = pending, queued, stales
				}
	
			// Handle inactive account transaction eviction
			// 处理非活动帐户交易驱逐
			case <-evict.C:
				pool.mu.Lock()
				for addr := range pool.queue {
					// Skip local transactions from the eviction mechanism
					if pool.locals.contains(addr) {
						continue
					}
					// Any non-locals old enough should be removed
					if time.Since(pool.beats[addr]) > pool.config.Lifetime {
						for _, tx := range pool.queue[addr].Flatten() {
							pool.removeTx(tx.Hash(), true)
						}
					}
				}
				pool.mu.Unlock()
	
			// Handle local transaction journal rotation
			//处理本地事务日志轮换
			case <-journal.C:
				if pool.journal != nil {
					pool.mu.Lock()
					if err := pool.journal.rotate(pool.local()); err != nil {
						log.Warn("Failed to rotate local tx journal", "err", err)
					}
					pool.mu.Unlock()
				}
			}
		}
	}

