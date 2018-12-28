# txpool #

----------
如果可处理，则add验证新事务并将其状态设置为pending。 如有必要，它还会更新本地存储的nonce。

	func (self *TxPool) add(ctx context.Context, tx types.RPTransaction) error {
		hash := tx.Hash()
	
		// 已知得事务
		if self.pending[hash] != nil {
			return fmt.Errorf("Known transaction (%x)", hash[:4])
		}
		// 检查事务地址的nonce,余额等是否有效
		err := self.validateTx(ctx, tx)
		if err != nil {
			return err
		}
	
		// 加入pending队列
		if _, ok := self.pending[hash]; !ok {
			self.pending[hash] = tx
	
			nonce := tx.Nonce() + 1
	
			addr, _ := types.Sender(self.signer, tx)
			if nonce > self.nonce[addr] {
				self.nonce[addr] = nonce
			}
	
			// Notify the subscribers. This event is posted in a goroutine
			// because it's possible that somewhere during the post "Remove transaction"
			// gets called which will then wait for the global tx pool lock and deadlock.
			go self.txFeed.Send(core.NewTxsEvent{Txs: types.RPTransactions{tx}})
		}
	
		// Print a log message if low enough level is set
		log.Debug("Pooled new transaction", "hash", hash, "from", log.Lazy{Fn: func() common.Address { from, _ := types.Sender(self.signer, tx); return from }}, "to", tx.To())
		return nil
	}

promoteTx向待处理（可处理的）事务列表添加一个事务，并返回它是否已插入或之前的已经变得更好了。

	func (pool *TxPool) promoteTx(addr common.Address, hash common.Hash, tx types.Transaction) bool {
		// Try to insert the transaction into the pending queue
		// 尝试将事务插入挂起队列
		if pool.pending[addr] == nil {
			// 严格按nonce排序
			pool.pending[addr] = newTxList(true)
		}
		list := pool.pending[addr]
	
		inserted, old := list.Add(tx, pool.config.PriceBump)
		if !inserted { //如果没插入，也就意味着旧得事务更好
			// An older transaction was better, discard this
			// 将事务忽略
			pool.all.Remove(hash)
			pool.priced.Removed()
	
			pendingDiscardCounter.Inc(1)
			return false
		}
		// Otherwise discard any previous transaction and mark this
		// 否则丢弃任何先前的交易并标记
		if old != nil { //删除旧事务
			pool.all.Remove(old.Hash())
			pool.priced.Removed()
	
			pendingReplaceCounter.Inc(1)
		}
		// Failsafe to work around direct pending inserts (tests)
		if pool.all.Get(hash) == nil {
			pool.all.Add(tx)
			pool.priced.Put(tx)
		}
		// Set the potentially new pending nonce and notify any subsystems of the new tx
		// 设置潜在的新挂起的nonce并通知新tx的任何子系统
		pool.beats[addr] = time.Now() //更新地址心跳时间
		pool.pendingState.SetNonce(addr, tx.Nonce()+1)//更新这个地址得nonce
	
		return true
	}



validateTx根据共识规则检查事务是否有效

	func (pool *TxPool) validateTx(ctx context.Context, tx *types.Transaction) error {
		// Validate sender
		var (
			from common.Address
			err  error
		)
	
		// Validate the transaction sender and it's sig. Throw
		// if the from fields is invalid.
		// 验证交易发件人及其签名。 如果from字段无效则抛出。
		if from, err = types.Sender(pool.signer, tx); err != nil {
			return core.ErrInvalidSender
		}
		// Last but not least check for nonce errors
	
		// 当前的数据库状态
		currentState := pool.currentState(ctx)
		// 判断数据库中存储的该地址nonce是否大于当前事务的nonce
		if n := currentState.GetNonce(from); n > tx.Nonce() {
			return core.ErrNonceTooLow
		}
	
		// Check the transaction doesn't exceed the current
		// block limit gas.
		// 检查事务是否超过当前的块gas限制
		header := pool.chain.GetHeaderByHash(pool.head)
		if header.GasLimit < tx.Gas() {
			return core.ErrGasLimit
		}
	
		// Transactions can't be negative. This may never happen
		// using RLP decoded transactions but may occur if you create
		// a transaction using the RPC for example.
		// 交易的值不能是负数的。 使用RLP解码的事务可能永远不会发生这种情况，但如果您使用RPC创建事务，则可能会发生这种情况。
		if tx.Value().Sign() < 0 {
			return core.ErrNegativeValue
		}
	
		// Transactor should have enough funds to cover the costs
		// cost == V + GP * GL
		// 检查余额是否足够
		if b := currentState.GetBalance(from); b.Cmp(tx.Cost()) < 0 {
			return core.ErrInsufficientFunds
		}
	
		// Should supply enough intrinsic gas
		// 应该供应足够的固有气体
		gas, err := core.IntrinsicGas(tx.Data(), tx.To() == nil, pool.homestead)
		if err != nil {
			return err
		}
		if tx.Gas() < gas {
			return core.ErrIntrinsicGas
		}
		return currentState.Error()
	}