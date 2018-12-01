# blockchain #

----------

BlockChain表示给定具有genesis块的数据库的规范链。 BlockChain管理链的导入，还原，重组。

将块导入块链, 需要经过两阶段Validator定义的规则集。 使用Processor来对区块的交易进行处理. 状态的验证是第二阶段的验证器。 失败会导致导入中止。

BlockChain还有助于从包含在数据库中的任何**链中返回块以及代表规范链的块。 重要的是要注意GetBlock可以返回任何块，并且不需要包含在规范的中，因为GetBlockByNumber始终表示规范链。

## 数据库存储结构 ##

----------


<table>
<thead>
<tr>
<th>key</th>
<th>value</th>
<th>说明</th>
<th>插入</th>
<th>删除</th>
</tr>
</thead>
<tbody>
<tr>
<td>"LastHeader"</td>
<td>hash</td>
<td>最新的区块头 HeaderChain中使用</td>
<td>当区块被认为是当前最新的一个规范的区块链头</td>
<td>当有了更新的区块链头或者是分叉的兄弟区块链替代了它</td>
</tr>
<tr>
<td>"LastBlock"</td>
<td>hash</td>
<td>最新的区块头 BlockChain中使用</td>
<td>当区块被认为是当前最新的一个规范的区块链头</td>
<td>当有了更新的区块链头或者是分叉的兄弟区块链替代了它</td>
</tr>
<tr>
<td>"LastFast"</td>
<td>hash</td>
<td>最新的区块头 BlockChain中使用</td>
<td>当区块被认为是当前最新的规范的区块链头</td>
<td>当有了更新的区块链头或者是分叉的兄弟区块链替代了它</td>
</tr>
<tr>
<td>"h"+num+"n"</td>
<td>hash</td>
<td>用来存储规范的区块链的高度和区块头的hash值 在HeaderChain中使用</td>
<td>当区块在规范的区块链中</td>
<td>当区块不在规范的区块链中</td>
</tr>
<tr>
<td>"h" + num + hash + "t"</td>
<td>td</td>
<td>总难度</td>
<td>WriteBlockAndState当验证并执行完一个区块之后(不管是不是规范的)</td>
<td>SetHead方法会调用。这种方法只会在两种情况下被调用，1是当前区块链包含了badhashs，需要删除所有从badhashs开始的区块， 2.是当前区块的状态错误，需要Reset到genesis。</td>
</tr>
<tr>
<td>"H" + hash</td>
<td>num</td>
<td>区块的高度 在HeaderChain中使用</td>
<td>WriteBlockAndState当验证并执行完一个区块之后</td>
<td>SetHead中被调用，同上</td>
</tr>
<tr>
<td>"b" + num + hash</td>
<td>block body</td>
<td>区块数据</td>
<td>WriteBlockAndState or InsertReceiptChain</td>
<td>SetHead中被删除，同上</td>
</tr>
<tr>
<td>"r" + num + hash</td>
<td>block receipts</td>
<td>区块收据</td>
<td>WriteBlockAndState or InsertReceiptChain</td>
<td>同上</td>
</tr>
<tr>
<td>"l" + txHash</td>
<td>{hash,num,TxIndex</td>
<td>交易hash可以找到区块和交易</td>
<td>当区块加入规范的区块链</td>
<td>当区块从规范的区块链移除</td>
</tr>
</tbody>
</table>

----------


blockchain结构体


	type BlockChain struct {
		chainConfig *params.ChainConfig // 链和网络的配置
		cacheConfig *CacheConfig        // 用于修剪的缓存配置
	
		db     ethdb.Database // 用于存储最终内容的底层持久性数据库
		triegc *prque.Prque   // 优先级队列映射块编号以尝试gc
		gcproc time.Duration  // 累积trie丢弃的规范块处理
	
		hc            *HeaderChain  //只包含了区块头的链
		rmLogsFeed    event.Feed	//消息通知组件
		chainFeed     event.Feed
		chainSideFeed event.Feed
		chainHeadFeed event.Feed
		logsFeed      event.Feed
		scope         event.SubscriptionScope
		genesisBlock  *types.Block //创世块
	
		mu      sync.RWMutex // 区块链操作的全局锁
		chainmu sync.RWMutex // 插入锁
		procmu  sync.RWMutex // 区块处理锁
	
		checkpoint       int          // 检查点计数
		currentBlock     atomic.Value // 当前区块链的块头
		currentFastBlock atomic.Value // 当前快速同步链的块头
	
		stateCache    state.Database // 状态数据库状态缓存
		bodyCache     *lru.Cache     // 最近区块块体的缓存
		bodyRLPCache  *lru.Cache     // 最近区块块体的缓存（rlp形式）
		receiptsCache *lru.Cache     // 最近的每隔块的收据缓存
		blockCache    *lru.Cache     // 最近的整个块的缓存
		futureBlocks  *lru.Cache     // 暂时还没加入到链中的块
	
		quit    chan struct{} // blockchain 关闭通道
		running int32         // running must be called atomically
		// procInterrupt must be atomically called
		procInterrupt int32          // 块处理的终端信号器
		wg            sync.WaitGroup // 块处理等待组
	
		engine    consensus.Engine
		processor Processor // 块处理器接口
		validator Validator // 块和状态验证器接口
		vmConfig  vm.Config // 虚拟机配置
	
		badBlocks      *lru.Cache              // 错误区块缓存
		shouldPreserve func(*types.Block) bool // 用于确定是否应保留给定块的函数。
	}


NewBlockChain

NewBlockChain使用数据库中可用的信息返回完全初始化的块链。 它初始化默认的以太坊验证器和处理器。


	func NewBlockChain(db ethdb.Database, cacheConfig *CacheConfig, chainConfig *params.ChainConfig, engine consensus.Engine, vmConfig vm.Config, shouldPreserve func(block *types.Block) bool) (*BlockChain, error) {
		//初始化修剪缓存配置
		if cacheConfig == nil {
			cacheConfig = &CacheConfig{
				TrieNodeLimit: 256 * 1024 * 1024,
				TrieTimeLimit: 5 * time.Minute,
			}
		}
		//缓存建立
		bodyCache, _ := lru.New(bodyCacheLimit)
		bodyRLPCache, _ := lru.New(bodyCacheLimit)
		receiptsCache, _ := lru.New(receiptsCacheLimit)
		blockCache, _ := lru.New(blockCacheLimit)
		futureBlocks, _ := lru.New(maxFutureBlocks)
		badBlocks, _ := lru.New(badBlockLimit)
	
		bc := &BlockChain{
			chainConfig:    chainConfig,
			cacheConfig:    cacheConfig,
			db:             db,
			triegc:         prque.New(nil), //-todo
			stateCache:     state.NewDatabase(db),
			quit:           make(chan struct{}),
			shouldPreserve: shouldPreserve,
			bodyCache:      bodyCache,
			bodyRLPCache:   bodyRLPCache,
			receiptsCache:  receiptsCache,
			blockCache:     blockCache,
			futureBlocks:   futureBlocks,
			engine:         engine,
			vmConfig:       vmConfig,
			badBlocks:      badBlocks,
		}
		//设定验证器和处理器
		bc.SetValidator(NewBlockValidator(chainConfig, bc, engine))
		bc.SetProcessor(NewStateProcessor(chainConfig, bc, engine))
	
		var err error
		//建立块头链
		bc.hc, err = NewHeaderChain(db, chainConfig, engine, bc.getProcInterrupt)
		if err != nil {
			return nil, err
		}
		//创世块
		bc.genesisBlock = bc.GetBlockByNumber(0)
		if bc.genesisBlock == nil {
			return nil, ErrNoGenesis
		}
		//从数据库加载最后一个已知的链状态
		if err := bc.loadLastState(); err != nil {
			return nil, err
		}
		// 检查块哈希的当前状态，并确保我们的链中没有任何坏块
		//手动设置的哈希值，通常是硬分叉带来的影响
		for hash := range BadHashes {
			if header := bc.GetHeaderByHash(hash); header != nil {
				// 获取与坏块头号码对应的规范块
				headerByNumber := bc.GetHeaderByNumber(header.Number.Uint64())
				// 确保这个对应的规范块在当前的链中
				if headerByNumber != nil && headerByNumber.Hash() == header.Hash() {
					log.Error("Found bad hash, rewinding chain", "number", header.Number, "hash", header.ParentHash)
					bc.SetHead(header.Number.Uint64() - 1) //包含坏块，所以要回滚到坏块之前的高度
					log.Error("Chain rewind was successful, resuming normal operation")
				}
			}
		}
		// 取得这个特定状态的所有权
		go bc.update()
		return bc, nil
	}


loadLastState

从数据库中获取最后知道的链的状态，这个方法假设已经获取到锁了。

	func (bc *BlockChain) loadLastState() error {
		// Restore the last known head block
		// 恢复最后一个已知的头部块
		head := rawdb.ReadHeadBlockHash(bc.db)  //拿到最后一个块的哈希
		if head == (common.Hash{}) {
			// Corrupt or empty database, init from scratch
			log.Warn("Empty database, resetting chain")
			return bc.Reset()
		}
		// Make sure the entire head block is available
		// 用块头哈希拿到当前最后一个块
		currentBlock := bc.GetBlockByHash(head)
		if currentBlock == nil {
			// Corrupt or empty database, init from scratch
			log.Warn("Head block missing, resetting chain", "hash", head)
			return bc.Reset()
		}
		// Make sure the state associated with the block is available
		// 确定这个块的状态是否正确
		if _, err := state.New(currentBlock.Root(), bc.stateCache); err != nil {
			// Dangling block without a state associated, init from scratch
			log.Warn("Head state missing, repairing chain", "number", currentBlock.Number(), "hash", currentBlock.Hash())
			if err := bc.repair(&currentBlock); err != nil {
				return err
			}
		}
		// Everything seems to be fine, set as the head block
		// 将其设为当前块
		bc.currentBlock.Store(currentBlock)
	
		// Restore the last known head header
		currentHeader := currentBlock.Header()
		if head := rawdb.ReadHeadHeaderHash(bc.db); head != (common.Hash{}) {
			if header := bc.GetHeaderByHash(head); header != nil {
				currentHeader = header
			}
		}
		// 设置块头
		bc.hc.SetCurrentHeader(currentHeader)
	
		// Restore the last known head fast block
		// 重置快速同步块头
		bc.currentFastBlock.Store(currentBlock)
		if head := rawdb.ReadHeadFastBlockHash(bc.db); head != (common.Hash{}) {
			if block := bc.GetBlockByHash(head); block != nil {
				bc.currentFastBlock.Store(block)
			}
		}
	
		// Issue a status log for the user
		// 通知用户最新的状态，包括块头，块，难度等
		currentFastBlock := bc.CurrentFastBlock()
	
		headerTd := bc.GetTd(currentHeader.Hash(), currentHeader.Number.Uint64())
		blockTd := bc.GetTd(currentBlock.Hash(), currentBlock.NumberU64())
		fastTd := bc.GetTd(currentFastBlock.Hash(), currentFastBlock.NumberU64())
	
		log.Info("Loaded most recent local header", "number", currentHeader.Number, "hash", currentHeader.Hash(), "td", headerTd, "age", common.PrettyAge(time.Unix(currentHeader.Time.Int64(), 0)))
		log.Info("Loaded most recent local full block", "number", currentBlock.Number(), "hash", currentBlock.Hash(), "td", blockTd, "age", common.PrettyAge(time.Unix(currentBlock.Time().Int64(), 0)))
		log.Info("Loaded most recent local fast block", "number", currentFastBlock.Number(), "hash", currentFastBlock.Hash(), "td", fastTd, "age", common.PrettyAge(time.Unix(currentFastBlock.Time().Int64(), 0)))
	
		return nil
	}



//定时处理相对来说以后的块


	func (bc *BlockChain) update() {
		futureTimer := time.NewTicker(5 * time.Second)
		defer futureTimer.Stop()
		for {
			select {
			case <-futureTimer.C:
				bc.procFutureBlocks()
			case <-bc.quit:
				return
			}
		}
	}


	func (bc *BlockChain) procFutureBlocks() {
		blocks := make([]*types.Block, 0, bc.futureBlocks.Len())
		for _, hash := range bc.futureBlocks.Keys() {
			if block, exist := bc.futureBlocks.Peek(hash); exist {
				blocks = append(blocks, block.(*types.Block))
			}
		}
		if len(blocks) > 0 {
			types.BlockBy(types.Number).Sort(blocks)
	
			// Insert one by one as chain insertion needs contiguous ancestry between blocks
			for i := range blocks {
				bc.InsertChain(blocks[i : i+1])
			}
		}
	}

Reset()清除整个区块链，将其恢复到其初始状态。

	func (bc *BlockChain) Reset() error {
		return bc.ResetWithGenesisBlock(bc.genesisBlock)
	}

	func (bc *BlockChain) ResetWithGenesisBlock(genesis *types.Block) error {
		// Dump the entire block chain and purge the caches
		if err := bc.SetHead(0); err != nil {
			return err
		}
		bc.mu.Lock()
		defer bc.mu.Unlock()
	
		// Prepare the genesis block and reinitialise the chain
		if err := bc.hc.WriteTd(genesis.Hash(), genesis.NumberU64(), genesis.Difficulty()); err != nil {
			log.Crit("Failed to write genesis block TD", "err", err)
		}
		rawdb.WriteBlock(bc.db, genesis)
	
		bc.genesisBlock = genesis
		bc.insert(bc.genesisBlock)
		bc.currentBlock.Store(bc.genesisBlock)
		bc.hc.SetGenesis(bc.genesisBlock.Header())
		bc.hc.SetCurrentHeader(bc.genesisBlock.Header())
		bc.currentFastBlock.Store(bc.genesisBlock)
	
		return nil
	}

SetHead 将本地链回溯到新的块头，也就是说让高度为head的块成为当前链的最后一个块，

	func (bc *BlockChain) SetHead(head uint64) error {
		log.Warn("Rewinding blockchain", "target", head)
	
		bc.mu.Lock()
		defer bc.mu.Unlock()
	
		// Rewind the header chain, deleting all block bodies until then
		delFn := func(db rawdb.DatabaseDeleter, hash common.Hash, num uint64) {
			rawdb.DeleteBody(db, hash, num)
		}
		// 设置块头
		bc.hc.SetHead(head, delFn)
	
		currentHeader := bc.hc.CurrentHeader()
	
		// Clear out any stale content from the caches
		// 清楚缓存中的旧内容
		bc.bodyCache.Purge()
		bc.bodyRLPCache.Purge()
		bc.receiptsCache.Purge()
		bc.blockCache.Purge()
		bc.futureBlocks.Purge()
	
		// Rewind the block chain, ensuring we don't end up with a stateless head block
		// 回放区块链，确保我们不会成为无状态头块
	
		//将当前块存储为设定的块头的块
		if currentBlock := bc.CurrentBlock(); currentBlock != nil && currentHeader.Number.Uint64() < currentBlock.NumberU64() {
			bc.currentBlock.Store(bc.GetBlock(currentHeader.Hash(), currentHeader.Number.Uint64()))
		}
		if currentBlock := bc.CurrentBlock(); currentBlock != nil {
			// 给定的树创建新的状态
			if _, err := state.New(currentBlock.Root(), bc.stateCache); err != nil {
				// Rewound state missing, rolled back to before pivot, reset to genesis
				// 无法回溯状态那么回到创世块
				bc.currentBlock.Store(bc.genesisBlock)
			}
		}
		// Rewind the fast block in a simpleton way to the target head
		if currentFastBlock := bc.CurrentFastBlock(); currentFastBlock != nil && currentHeader.Number.Uint64() < currentFastBlock.NumberU64() {
			bc.currentFastBlock.Store(bc.GetBlock(currentHeader.Hash(), currentHeader.Number.Uint64()))
		}
		// If either blocks reached nil, reset to the genesis state
		if currentBlock := bc.CurrentBlock(); currentBlock == nil {
			bc.currentBlock.Store(bc.genesisBlock)
		}
		if currentFastBlock := bc.CurrentFastBlock(); currentFastBlock == nil {
			bc.currentFastBlock.Store(bc.genesisBlock)
		}
		currentBlock := bc.CurrentBlock()
		currentFastBlock := bc.CurrentFastBlock()
	
		rawdb.WriteHeadBlockHash(bc.db, currentBlock.Hash())
		rawdb.WriteHeadFastBlockHash(bc.db, currentFastBlock.Hash())
	
		return bc.loadLastState()
	}


headerchain.go--SetHead 新头部上方的所有内容都将被删除，新的块头将被设置。

	func (hc *HeaderChain) SetHead(head uint64, delFn DeleteCallback) {
		height := uint64(0)
		// 获取当前HeaderChain的块头
		if hdr := hc.CurrentHeader(); hdr != nil {
			height = hdr.Number.Uint64() //转换为整数
		}
		batch := hc.chainDb.NewBatch()
		// 判断要设置的高度是否小于目前的高度
		// 迭代删除块
		for hdr := hc.CurrentHeader(); hdr != nil && hdr.Number.Uint64() > head; hdr = hc.CurrentHeader() {
			hash := hdr.Hash()
			num := hdr.Number.Uint64()
			if delFn != nil {
				delFn(batch, hash, num) //删除块体
			}
			rawdb.DeleteHeader(batch, hash, num) //删除块头
			rawdb.DeleteTd(batch, hash, num) //删除难度
			// 将当前块头设置为前一块
			hc.currentHeader.Store(hc.GetHeader(hdr.ParentHash, hdr.Number.Uint64()-1))
		}
		// Roll back the canonical chain numbering
		// 删除规范链编号
		for i := height; i > head; i-- {
			rawdb.DeleteCanonicalHash(batch, i)
		}
		batch.Write()
	
		// Clear out any stale content from the caches
		// 清楚缓存中得旧内容
		hc.headerCache.Purge()
		hc.tdCache.Purge()
		hc.numberCache.Purge()
	
		if hc.CurrentHeader() == nil {
			hc.currentHeader.Store(hc.genesisHeader)
		}
		hc.currentHeaderHash = hc.CurrentHeader().Hash()
		//
		rawdb.WriteHeadHeaderHash(hc.chainDb, hc.currentHeaderHash)
	}

Rollback从数据库中删除不确定有效的链接链。

	func (bc *BlockChain) Rollback(chain []common.Hash) {
		bc.mu.Lock()
		defer bc.mu.Unlock()
		// 如果当前块与chain中的块哈希相同那么就将链回滚到上一块
		// 也就是说这个chain中存放的事无效块的哈希
		for i := len(chain) - 1; i >= 0; i-- {
			hash := chain[i]
	
			currentHeader := bc.hc.CurrentHeader()
			if currentHeader.Hash() == hash {
			
				bc.hc.SetCurrentHeader(bc.GetHeader(currentHeader.ParentHash, currentHeader.Number.Uint64()-1))
			}
			if currentFastBlock := bc.CurrentFastBlock(); currentFastBlock.Hash() == hash {
				newFastBlock := bc.GetBlock(currentFastBlock.ParentHash(), currentFastBlock.NumberU64()-1)
				bc.currentFastBlock.Store(newFastBlock)
				rawdb.WriteHeadFastBlockHash(bc.db, newFastBlock.Hash())
			}
			if currentBlock := bc.CurrentBlock(); currentBlock.Hash() == hash {
				newBlock := bc.GetBlock(currentBlock.ParentHash(), currentBlock.NumberU64()-1)
				bc.currentBlock.Store(newBlock)
				rawdb.WriteHeadBlockHash(bc.db, newBlock.Hash())
			}
		}
	}