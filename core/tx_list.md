# tx_list #

----------

txList是属于帐户的事务的“列表”，按帐户nonce排序。 可用于存储可执行/挂起队列的连续事务和存储不可执行/未来队列的间隙事务，并且具有微小的行为改变。

	type txList struct {
		// 决定是否按照nonces严格排列
		strict bool         // Whether nonces are strictly continuous or not
		// 事务堆排序map
		txs    *txSortedMap // Heap indexed sorted hash map of the transactions
	
		//最高成本交易的价格（仅在超出余额时重置）
		costcap *big.Int // Price of the highest costing transaction (reset only if exceeds balance)
		//最高支出交易的燃气限制（仅在超过限额时重置）
		gascap  uint64   // Gas limit of the highest spending transaction (reset only if exceeds block limit)
	}



添加尝试将新事务插入列表，返回事务是否被接受，如果是，则替换之前的任何事务。 如果新事务被接受到列表中，则列表的成本和气体阈值也可能会更新。

	func (l *txList) Add(tx types.Transaction, priceBump uint64) (bool, types.Transaction) {
		// If there's an older better transaction, abort
		// 如果有较旧的更好的交易，则中止
		old := l.txs.Get(tx.Nonce())
		if old != nil {
			threshold := new(big.Int).Div(new(big.Int).Mul(old.GasPrice(), big.NewInt(100+int64(priceBump))), big.NewInt(100))
			// Have to ensure that the new gas price is higher than the old gas
			// price as well as checking the percentage threshold to ensure that
			// this is accurate for low (Wei-level) gas price replacements
			// 必须确保新的gas价格高于旧gas价格以及检查百分比阈值，以确保这对于低gas价格替换是准确的
			if old.GasPrice().Cmp(tx.GasPrice()) >= 0 || threshold.Cmp(tx.GasPrice()) > 0 {// 旧得更好
				return false, nil
			}
		}
		// Otherwise overwrite the old transaction with the current one
		// 否则用当前事务覆盖旧事务
		l.txs.Put(tx)
		// costcap作用是什么-todo
		if cost := tx.Cost(); l.costcap.Cmp(cost) < 0 {
			l.costcap = cost
		}
		if gas := tx.Gas(); l.gascap < gas {
			l.gascap = gas
		}
		return true, old
	}