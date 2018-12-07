# journal #

----------

journalEntry是状态更改日志中的修改条目，可以按需还原。

journalEntry接口
	
	type journalEntry interface {
		// revert undoes the changes introduced by this journal entry.
		// 撤消此journalEntry的更改。
		revert(*StateDB)
	
		// dirtied returns the Ethereum address modified by this journal entry.
		// 返回此journalEntry修改的以太坊地址
		dirtied() *common.Address
	}


journal包含自上次状态提交以来应用的状态修改列表。 跟踪这些是在执行异常或恢复请求的情况下能够恢复的。

	type journal struct {
		//跟踪当前的变化
		entries []journalEntry         // Current changes tracked by the journal
		//更改的账户的地址以及更改的次数
		dirties map[common.Address]int // Dirty accounts and the number of changes
	}


append在更改日志的末尾插入一个新的修改条目

	func (j *journal) append(entry journalEntry) {
		j.entries = append(j.entries, entry)
		if addr := entry.dirtied(); addr != nil {
			j.dirties[*addr]++
		}
	}

修改行为


	type (
		// Changes to the account trie.
		// 账户树的更改
	
		//创建状态对象
		createObjectChange struct {
			account *common.Address
		}
		//重置对象
		resetObjectChange struct {
			prev *stateObject
		}
		//自杀
		suicideChange struct {
			account     *common.Address
			prev        bool // whether account had already suicided
			prevbalance *big.Int
		}
	
		// Changes to individual accounts.
		// 个体账户的改变
	
		//余额改变
		balanceChange struct {
			account *common.Address
			prev    *big.Int
		}
		// nonce改变
		nonceChange struct {
			account *common.Address
			prev    uint64
		}
		// 存储空间改变
		storageChange struct {
			account       *common.Address
			key, prevalue common.Hash
		}
		// 代码改变
		codeChange struct {
			account            *common.Address
			prevcode, prevhash []byte
		}
	
		// Changes to other state values.
		// 其他状态值改变
	
		// 返还金额改变
		refundChange struct {
			prev uint64
		}
		// 增加日志
		addLogChange struct {
			txhash common.Hash
		}
		//原象改变
		addPreimageChange struct {
			hash common.Hash
		}
		//
		touchChange struct {
			account   *common.Address
			prev      bool
			prevDirty bool
		}
	)