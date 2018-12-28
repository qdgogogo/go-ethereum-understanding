# tx_cacher #

----------

senderCacher 是当前事务发送者得还原和缓存

	var senderCacher = newTxSenderCacher(runtime.NumCPU())

inc字段定义每次恢复后要跳过的事务数，用于将相同的底层输入数组提供给不同的线程，但确保它们快速处理早期事务。
	
	type txSenderCacherRequest struct {
		signer types.Signer
		txs    []*types.Transaction
		inc    int
	}

newTxSenderCacher创建一个新的事务发送方后台cacher，并在构造时启动GOMAXPROCS允许的尽可能多的处理goroutine。

	func newTxSenderCacher(threads int) *txSenderCacher {
		cacher := &txSenderCacher{
			tasks:   make(chan *txSenderCacherRequest, threads),
			threads: threads,
		}
		for i := 0; i < threads; i++ {
			go cacher.cache()
		}
		return cacher
	}

txSenderCacherRequest是使用特定签名方案恢复事务发件人并将其缓存到事务本身的请求。

	func (cacher *txSenderCacher) cache() {
		for task := range cacher.tasks {
			for i := 0; i < len(task.txs); i += task.inc {// 每次都跳过inc个事务
				types.Sender(task.signer, task.txs[i]) //更新并缓存事务发送者的地址
			}
		}
	}



recoverFromBlocks从一批块中恢复发送者并将它们缓存回相同的数据结构中
	
	func (cacher *txSenderCacher) recoverFromBlocks(signer types.Signer, blocks []*types.Block) {
		count := 0 //整合这些块中所有事务
		for _, block := range blocks {
			count += len(block.Transactions())
		}
		txs := make([]*types.Transaction, 0, count)
		for _, block := range blocks {
			txs = append(txs, block.Transactions()...)
		}
		cacher.recover(signer, txs)
	}

recoverFromBlocks从一批事务中恢复发送者并将它们缓存回相同的数据结构中

	func (cacher *txSenderCacher) recover(signer types.Signer, txs []*types.Transaction) {
		// If there's nothing to recover, abort
	
		if len(txs) == 0 {
			return
		}
		// Ensure we have meaningful task sizes and schedule the recoveries
		// 确保我们具有有意义的任务大小并安排恢复
		tasks := cacher.threads
		if len(txs) < tasks*4 {
			tasks = (len(txs) + 3) / 4
		}
		for i := 0; i < tasks; i++ {
			cacher.tasks <- &txSenderCacherRequest{
				signer: signer,
				// 这里这么做得原因时能够平均快速的恢复
				// 例如cpu是4那么，4个tasks每次每个task都会跳过inc(4)个事务，而且是并行执行的
				txs:    txs[i:],
				inc:    tasks,
			}
		}
	}


