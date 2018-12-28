# fetcher #

----------
fetcher包含基于块通知的同步。当我们接收到NewBlockHashesMsg消息得时候，我们只收到了很多Block的hash值。 需要通过hash值来同步区块，然后更新本地区块链。 fetcher就提供了这样的功能。

	type Fetcher struct {
		// Various event channels
		// 事件通道
		notify chan *announce
		inject chan *inject
	
		blockFilter  chan chan []*types.Block//块过滤器
		headerFilter chan chan *headerFilterTask//header过滤器
		bodyFilter   chan chan *bodyFilterTask//body过滤器
	
		done chan common.Hash	//完成
		quit chan struct{}
	
		// Announce states
		// key是peer的名字， value是announce的count
		announces  map[string]int              // Per peer announce counts to prevent memory exhaustion
		// 等待调度fetching的announce
		announced  map[common.Hash][]*announce // Announced blocks, scheduled for fetching
		// 正在fetching的announce
		fetching   map[common.Hash]*announce   // Announced blocks, currently fetching
		// 已经获取区块头的，等待获取区块body
		fetched    map[common.Hash][]*announce // Blocks with headers fetched, scheduled for body retrieval
		// 头和体都已经获取完成的announce
		completing map[common.Hash]*announce   // Blocks with headers, currently body-completing
	
		// Block cache
		// 包含了import操作的队列(按照区块号排列)
		queue  *prque.Prque            // Queue containing the import operations (block number sorted)
		// key是peer，value是block数量。
		queues map[string]int          // Per peer block counts to prevent memory exhaustion
		// 已经放入队列的区块。 为了去重
		queued map[common.Hash]*inject // Set of already queued blocks (to dedupe imports)
	
		// Callbacks
		// 回调函数
		getBlock       blockRetrievalFn   // Retrieves a block from the local chain
		verifyHeader   headerVerifierFn   // Checks if a block's headers have a valid proof of work
		broadcastBlock blockBroadcasterFn // Broadcasts a block to connected peers
		chainHeight    chainHeightFn      // Retrieves the current chain's height
		insertChain    chainInsertFn      // Injects a batch of blocks into the chain
		dropPeer       peerDropFn         // Drops a peer for misbehaving
	
		// Testing hooks
		// 测试
		announceChangeHook func(common.Hash, bool) // Method to call upon adding or deleting a hash from the announce list
		queueChangeHook    func(common.Hash, bool) // Method to call upon adding or deleting a block from the import queue
		fetchingHook       func([]common.Hash)     // Method to call upon starting a block (eth/61) or header (eth/62) fetch
		completingHook     func([]common.Hash)     // Method to call upon starting a block body fetch (eth/62)
		importedHook       func(*types.Block)      // Method to call upon successful block import (both eth/61 and eth/62)
	}

announce是网络中新块可用性的哈希通知。

	type announce struct {
		// 新区块的hash值
		hash   common.Hash   // Hash of the block being announced
		// 区块的高度值
		number uint64        // Number of the block being announced (0 = unknown | old protocol)
		// 重新组装的区块头
		header *types.Header // Header of the block partially reassembled (new protocol)
		time   time.Time     // Timestamp of the announcement
	
		//发起通知的对等方的标识符
		origin string // Identifier of the peer originating the notification
	
		//获取区块头的函数指针， 里面包含了peer的信息。就是说找谁要这个区块头
		fetchHeader headerRequesterFn // Fetcher function to retrieve the header of an announced block
		// 获取区块体的函数指针
		fetchBodies bodyRequesterFn   // Fetcher function to retrieve the body of an announced block
	}


启动基于通知的同步器，接受并处理哈希通知并阻止提取，直到请求终止。

	func (f *Fetcher) Start() {
		go f.loop()
	}



insert会生成一个新的goroutine来运行一个块插入到链中。 如果块的编号与当前导入阶段的高度相同，则相应地更新阶段状态。

	func (f *Fetcher) insert(peer string, block *types.Block) {
		hash := block.Hash()
	
		// Run the import on a new thread
		// 在新线程上运行导入
		log.Debug("Importing propagated block", "peer", peer, "number", block.Number(), "hash", hash)
		go func() {
			defer func() { f.done <- hash }()
	
			// If the parent's unknown, abort insertion
			// 如果父母未知，则插入中止
			parent := f.getBlock(block.ParentHash())
			if parent == nil {
				log.Debug("Unknown parent of propagated block", "peer", peer, "number", block.Number(), "hash", hash, "parent", block.ParentHash())
				return
			}
			// Quickly validate the header and propagate the block if it passes
			// 快速验证块头并传播块（如果它通过）
			switch err := f.verifyHeader(block.Header()); err {
			case nil:
				// All ok, quickly propagate to our peers
				// 检查通过并广播给peers
				propBroadcastOutTimer.UpdateSince(block.ReceivedAt)
				// 调用的是handler中的方法，并且将块整体传播
				go f.broadcastBlock(block, true)
	
			case consensus.ErrFutureBlock:
				// Weird future block, don't fail, but neither propagate
				// 奇怪的未来块，不要失败，但都不会传播
	
			default:
				// Something went very wrong, drop the peer
				// 出了点问题，放弃该peer
				log.Debug("Propagated block verification failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
				f.dropPeer(peer)
				return
			}
			// Run the actual import and log any issues
			// 实际的插入操作
			// 最终执行的是core\blockchain中的insertchain方法
			if _, err := f.insertChain(types.Blocks{block}); err != nil {
				log.Debug("Propagated block import failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
				return
			}
			// If import succeeded, broadcast the block
			propAnnounceOutTimer.UpdateSince(block.ReceivedAt)
			// 传播声明该块的哈希
			go f.broadcastBlock(block, false)
	
			// Invoke the testing hook if needed
			if f.importedHook != nil {
				f.importedHook(block)
			}
		}()
	}

rescheduleFetch

	func (f *Fetcher) rescheduleFetch(fetch *time.Timer) {
		// Short circuit if no blocks are announced
		// 没有块等待那么返回
		if len(f.announced) == 0 {
			return
		}
	
		// Otherwise find the earliest expiring announcement
		// 否则找到最早过期的声明
		earliest := time.Now()
		// 在这里迭代所有的声明找到最早声明的那个块
		for _, announces := range f.announced {
	
			if earliest.After(announces[0].time) {
				earliest = announces[0].time
			}
		}
		// arriveTimeout是一个被声明的块明确的被请求之前允许的时间
		// 也就是设置announce到期时间
		fetch.Reset(arriveTimeout - time.Since(earliest))
	}

enqueue 如果新的块被导入但是还没被看到，调度新的导入操作

	func (f *Fetcher) enqueue(peer string, block *types.Block) {
		hash := block.Hash()
	
		// Ensure the peer isn't DOSing us
		// 确保peer没有DOS攻击我们
		// 也就是确保一个peer发送的块的数量不超过一定值（64）
		count := f.queues[peer] + 1
		if count > blockLimit {
			log.Debug("Discarded propagated block, exceeded allowance", "peer", peer, "number", block.Number(), "hash", hash, "limit", blockLimit)
			propBroadcastDOSMeter.Mark(1)
			// 清除记录
			f.forgetHash(hash)
			return
		}
		// Discard any past or too distant blocks
		// 抛弃过去的过着太远的块
		if dist := int64(block.NumberU64()) - int64(f.chainHeight()); dist < -maxUncleDist || dist > maxQueueDist {
			log.Debug("Discarded propagated block, too far away", "peer", peer, "number", block.Number(), "hash", hash, "distance", dist)
			propBroadcastDropMeter.Mark(1)
			f.forgetHash(hash)
			return
		}
		// Schedule the block for future importing
		// 将块加入队列一边将来导入
		if _, ok := f.queued[hash]; !ok {
			op := &inject{
				origin: peer,
				block:  block,
			}
			f.queues[peer] = count
			f.queued[hash] = op
			// 放入队列
			f.queue.Push(op, -int64(block.NumberU64()))
			if f.queueChangeHook != nil {
				f.queueChangeHook(op.block.Hash(), true)
			}
			log.Debug("Queued propagated block", "peer", peer, "number", block.Number(), "hash", hash, "queued", f.queue.Size())
		}
	}