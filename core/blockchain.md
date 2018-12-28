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

insertChain将执行实际的链插入和事件聚合。 此方法作为单独方法存在的唯一原因是使用延迟语句使锁定更清晰。

	func (bc *BlockChain) insertChain(chain types.Blocks) (int, []interface{}, []*types.Log, error) {
		// Sanity check that we have something meaningful to import
		// 完整性检查
		if len(chain) == 0 {
			return 0, nil, nil, nil
		}
		// Do a sanity check that the provided chain is actually ordered and linked
		// 进行健全性检查，确保所提供的链实际已排序和链接
		for i := 1; i < len(chain); i++ {
			if chain[i].NumberU64() != chain[i-1].NumberU64()+1 || chain[i].ParentHash() != chain[i-1].Hash() {
				// Chain broke ancestry, log a message (programming error) and skip insertion
				log.Error("Non contiguous block insert", "number", chain[i].Number(), "hash", chain[i].Hash(),
					"parent", chain[i].ParentHash(), "prevnumber", chain[i-1].Number(), "prevhash", chain[i-1].Hash())
	
				return 0, nil, nil, fmt.Errorf("non contiguous insert: item %d is #%d [%x…], item %d is #%d [%x…] (parent [%x…])", i-1, chain[i-1].NumberU64(),
					chain[i-1].Hash().Bytes()[:4], i, chain[i].NumberU64(), chain[i].Hash().Bytes()[:4], chain[i].ParentHash().Bytes()[:4])
			}
		}
		// Pre-checks passed, start the full block imports
		// 开始导入整个块
		bc.wg.Add(1)
		defer bc.wg.Done()
	
		bc.chainmu.Lock()
		defer bc.chainmu.Unlock()
	
		// A queued approach to delivering events. This is generally
		// faster than direct delivery and requires much less mutex
		// acquiring.
		// 提供事件的排队方法。 这通常比直接交付更快，并且需要更少的互斥量获取。
		var (
			stats         = insertStats{startTime: mclock.Now()}
			events        = make([]interface{}, 0, len(chain))
			lastCanon     *types.Block
			coalescedLogs []*types.Log
		)
		// Start the parallel header verifier
		// 启动并行头验证程序
		headers := make([]*types.Header, len(chain))
		seals := make([]bool, len(chain))
	
		for i, block := range chain {
			headers[i] = block.Header()
			seals[i] = true
		}
		abort, results := bc.engine.VerifyHeaders(bc, headers, seals)
		defer close(abort)
	
		// Start a parallel signature recovery (signer will fluke on fork transition, minimal perf loss)
		// 启动并行签名恢复（签名者将在fork转换时出现错误，最小的性能损失）
		// 也就是将事务发送者的地址用签名恢复出来，并缓存
		senderCacher.recoverFromBlocks(types.MakeSigner(bc.chainConfig, chain[0].Number()), chain)
	
		// Iterate over the blocks and insert when the verifier permits
		// 迭代块并在验证者允许时插入
		for i, block := range chain {
			// If the chain is terminating, stop processing blocks
			// 如果链终止那么停止处理块
			if atomic.LoadInt32(&bc.procInterrupt) == 1 {
				log.Debug("Premature abort during blocks processing")
				break
			}
			// If the header is a banned one, straight out abort
			// 判断是否是坏的块头
			if BadHashes[block.Hash()] {
				bc.reportBlock(block, nil, ErrBlacklistedHash)
				return i, events, coalescedLogs, ErrBlacklistedHash
			}
			// Wait for the block's verification to complete
			//等待块验证完成
			bstart := time.Now()
	
			err := <-results
			if err == nil {
				// 验证块体
				err = bc.Validator().ValidateBody(block)
			}
			switch {
			case err == ErrKnownBlock:
				// Block and state both already known. However if the current block is below
				// this number we did a rollback and we should reimport it nonetheless.
				// 块和状态都已知。但是，如果当前块低于此数字，我们会进行回滚，但我们应该重新导入它。
				if bc.CurrentBlock().NumberU64() >= block.NumberU64() { //块已知，跳过
					stats.ignored++
					continue
				}
	
			case err == consensus.ErrFutureBlock: //这是将来的块
				// Allow up to MaxFuture second in the future blocks. If this limit is exceeded
				// the chain is discarded and processed at a later time if given.
				// 在未来的块中允许最多MaxFuture。 如果超过此限制，则链被丢弃并在稍后处理（如果给出）。
				// 也就是说这个块的时间戳超出现在事件一定的时间限制
				max := big.NewInt(time.Now().Unix() + maxTimeFutureBlocks)
				if block.Time().Cmp(max) > 0 {
					return i, events, coalescedLogs, fmt.Errorf("future block: %v > %v", block.Time(), max)
				}
				bc.futureBlocks.Add(block.Hash(), block)
				stats.queued++
				continue
	
			case err == consensus.ErrUnknownAncestor && bc.futureBlocks.Contains(block.ParentHash()):// 将来的块
				bc.futureBlocks.Add(block.Hash(), block)
				stats.queued++
				continue
	
			case err == consensus.ErrPrunedAncestor:// 该块状态不可用
				// Block competing with the canonical chain, store in the db, but don't process
				// until the competitor TD goes above the canonical TD
				// 该块与规范链竞争，存储在数据库中，但在竞争对手TD超出规范TD之前不进行处理
				currentBlock := bc.CurrentBlock()
				localTd := bc.GetTd(currentBlock.Hash(), currentBlock.NumberU64())
				externTd := new(big.Int).Add(bc.GetTd(block.ParentHash(), block.NumberU64()-1), block.Difficulty())
				if localTd.Cmp(externTd) > 0 { //新块难度比当前难度小，不是最优
					if err = bc.WriteBlockWithoutState(block, externTd); err != nil {
						return i, events, coalescedLogs, err
					}
					continue
				}
				// Competitor chain beat canonical, gather all blocks from the common ancestor
				// 该块比目前块优，收集他们的共同祖先
				var winner []*types.Block
	
				parent := bc.GetBlock(block.ParentHash(), block.NumberU64()-1)
				for !bc.HasState(parent.Root()) {
					winner = append(winner, parent)
					parent = bc.GetBlock(parent.ParentHash(), parent.NumberU64()-1)
				}
				for j := 0; j < len(winner)/2; j++ {
					winner[j], winner[len(winner)-1-j] = winner[len(winner)-1-j], winner[j]
				}
				// Import all the pruned blocks to make the state available
				// 导入所有已修剪的块以使状态可用
				bc.chainmu.Unlock()
				// 将这些优势块插入当前链
				// 递归
				_, evs, logs, err := bc.insertChain(winner)
				bc.chainmu.Lock()
				events, coalescedLogs = evs, logs
	
				if err != nil {
					return i, events, coalescedLogs, err
				}
	
			case err != nil:
				// 错误的块
				bc.reportBlock(block, nil, err)
				return i, events, coalescedLogs, err
			}
			// Create a new statedb using the parent block and report an
			// error if it fails.
			// 使用父块创建新的statedb，如果失败则报告错误。
			var parent *types.Block
			if i == 0 {
				parent = bc.GetBlock(block.ParentHash(), block.NumberU64()-1)
			} else {
				parent = chain[i-1]
			}
			// 用父块创建新的状态
			// 状态的构建过程
			state, err := state.New(parent.Root(), bc.stateCache)
			if err != nil {
				return i, events, coalescedLogs, err
			}
			// Process block using the parent state as reference point.
			// 使用父状态作为参考点的处理块。
			// 处理块中的状态
			// 调用了state_processor.go 里面的 Process方法
			receipts, logs, usedGas, err := bc.processor.Process(block, state, bc.vmConfig)
			if err != nil {
				bc.reportBlock(block, receipts, err)
				return i, events, coalescedLogs, err
			}
			// Validate the state using the default validator
			// 使用默认验证器验证状态
			// 二次验证,验证状态是否合法
			err = bc.Validator().ValidateState(block, parent, state, receipts, usedGas)
			if err != nil {
				bc.reportBlock(block, receipts, err)
				return i, events, coalescedLogs, err
			}
			proctime := time.Since(bstart)
	
			// Write the block to the chain and get the status.
			// 将块写入链并获取状态。
			//
			status, err := bc.WriteBlockWithState(block, receipts, state)
			if err != nil {
				return i, events, coalescedLogs, err
			}
			switch status {
			case CanonStatTy:// 插入了新的区块.
				log.Debug("Inserted new block", "number", block.Number(), "hash", block.Hash(), "uncles", len(block.Uncles()),
					"txs", len(block.Transactions()), "gas", block.GasUsed(), "elapsed", common.PrettyDuration(time.Since(bstart)))
	
				coalescedLogs = append(coalescedLogs, logs...)
				blockInsertTimer.UpdateSince(bstart)
				events = append(events, ChainEvent{block, block.Hash(), logs})
				lastCanon = block
	
				// Only count canonical blocks for GC processing time
				bc.gcproc += proctime
	
			case SideStatTy: // 插入了一个forked 区块
				log.Debug("Inserted forked block", "number", block.Number(), "hash", block.Hash(), "diff", block.Difficulty(), "elapsed",
					common.PrettyDuration(time.Since(bstart)), "txs", len(block.Transactions()), "gas", block.GasUsed(), "uncles", len(block.Uncles()))
	
				blockInsertTimer.UpdateSince(bstart)
				events = append(events, ChainSideEvent{block})
			}
			stats.processed++
			stats.usedGas += usedGas
	
			cache, _ := bc.stateCache.TrieDB().Size()
			stats.report(chain, i, cache)
		}
		// Append a single chain head event if we've progressed the chain
		// 如果我们生成了一个新的区块头, 而且最新的区块头等于lastCanon，那么我们公布一个新的 ChainHeadEvent
		if lastCanon != nil && bc.CurrentBlock().Hash() == lastCanon.Hash() {
			events = append(events, ChainHeadEvent{lastCanon})
		}
		return 0, events, coalescedLogs, nil
	}