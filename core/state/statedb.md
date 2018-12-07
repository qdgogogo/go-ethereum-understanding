# statedb #

----------


以太坊协议中的StateDB用于存储merkle trie中的任何内容。 StateDB负责缓存和存储嵌套状态。 这是检索的通用查询接口：

	type StateDB struct {
		db   Database	//数据库
		trie Trie		//Trie树
	
		// This map holds 'live' objects, which will get modified while processing a state transition.
		// 此映射包含“实时”对象，在处理状态转换时将对其进行修改。
		stateObjects      map[common.Address]*stateObject	//缓存对象
		stateObjectsDirty map[common.Address]struct{}		//缓存修改过的对象
	
		// DB error.
		// State objects are used by the consensus core and VM which are
		// unable to deal with database-level errors. Any error that occurs
		// during a database read is memoized here and will eventually be returned
		// by StateDB.Commit.
		// 共享核心和VM使用状态对象，它们无法处理数据库级错误。 在数据库读取期间发生的任何错误都会在此处进行记忆，并最终由StateDB.Commit返回。
		dbErr error
	
		// The refund counter, also used by state transitioning.
		// 退款计数，也用于状态转换
		refund uint64
	
		thash, bhash common.Hash	//当前的transaction hash 和block hash
		txIndex      int			//当前事务的索引序号
		logs         map[common.Hash][]*types.Log	//事务的hash作为key，对于的日志作为value
		logSize      uint		//日志大小
	
		preimages map[common.Hash][]byte	 // EVM计算的 SHA3->byte[]的映射关系
	
		// Journal of state modifications. This is the backbone of
		// Snapshot and RevertToSnapshot.
		// 状态修改日志。 这是Snapshot和RevertToSnapshot的支柱。
		journal        *journal
		validRevisions []revision
		nextRevisionId int
	
		lock sync.Mutex
	}


对于给定的trie创建一个新的状态
	
	func New(root common.Hash, db Database) (*StateDB, error) {
		tr, err := db.OpenTrie(root)
		if err != nil {
			return nil, err
		}
		return &StateDB{
			db:                db,
			trie:              tr,
			stateObjects:      make(map[common.Address]*stateObject),
			stateObjectsDirty: make(map[common.Address]struct{}),
			logs:              make(map[common.Hash][]*types.Log),
			preimages:         make(map[common.Hash][]byte),
			journal:           newJournal(),
		}, nil
	}

CreateAccount显式创建一个状态对象。 如果具有该地址的状态对象已存在，则余额将转入新帐户。

在EVM CREATE操作期间调用CreateAccount。 可能出现合同执行以下情况的情况：


1. 向sha(account ++ (nonce + 1))发送资金
1. tx_create（sha（account ++ nonce））（注意这个地址为1）


携带余额确保以太币不会消失


	func (self *StateDB) CreateAccount(addr common.Address) {
		new, prev := self.createObject(addr)
		if prev != nil {
			new.setBalance(prev.data.Balance)
		}
	}


创建一个新的状态对象。 如果存在具有给定地址的现有帐户，则将其覆盖并作为第二个返回值返回。

	func (self *StateDB) createObject(addr common.Address) (newobj, prev *stateObject) {
		// 获取现有地址的账户
		prev = self.getStateObject(addr)
		newobj = newObject(self, addr, Account{})
		newobj.setNonce(0) // sets the object to dirty
		// 如果为空则创建新对象，否则重置旧对象
		if prev == nil {
			self.journal.append(createObjectChange{account: &addr})
		} else {
			self.journal.append(resetObjectChange{prev: prev})
		}
		// 添加映射
		self.setStateObject(newobj)
		return newobj, prev
	}

检索地址给出的状态对象。 如果找不到则返回nil。

	func (self *StateDB) getStateObject(addr common.Address) (stateObject *stateObject) {
		// Prefer 'live' objects.
		// 返回已存在的对象
		if obj := self.stateObjects[addr]; obj != nil {
			if obj.deleted {
				return nil
			}
			return obj
		}
	
		// Load the object from the database.
		// 从数据库中加载对象
		enc, err := self.trie.TryGet(addr[:])
		if len(enc) == 0 {
			self.setError(err)
			return nil
		}
		var data Account
		// 将对象解码为账户
		if err := rlp.DecodeBytes(enc, &data); err != nil {
			log.Error("Failed to decode state object", "addr", addr, "err", err)
			return nil
		}
		// Insert into the live set.
		//将该对象插入实时对象集
		obj := newObject(self, addr, data)
		self.setStateObject(obj)
		return obj
	}




计算状态trie的当前根哈希值。 在事务之间调用它以获取进入事务收据的根哈希。

	func (s *StateDB) IntermediateRoot(deleteEmptyObjects bool) common.Hash {
		s.Finalise(deleteEmptyObjects)
		return s.trie.Hash()
	}

 通过删除自毁对象并清除日志和退款来最终确定状态。
	
	func (s *StateDB) Finalise(deleteEmptyObjects bool) {
		// 遍历每个地址的修改
		for addr := range s.journal.dirties {
			// 判断状态对象是否在实施对象中
			stateObject, exist := s.stateObjects[addr]
			if !exist {
				// ripeMD is 'touched' at block 1714175, in tx 0x1237f737031e40bcde4a8b7e717b2d15e3ecadfe49bb1bbc71ee9deb09c6fcf2
				// That tx goes out of gas, and although the notion of 'touched' does not exist there, the
				// touch-event will still be recorded in the journal. Since ripeMD is a special snowflake,
				// it will persist in the journal even though the journal is reverted. In this special circumstance,
				// it may exist in `s.journal.dirties` but not in `s.stateObjects`.
				// Thus, we can safely ignore it here
				continue
			}
	
			if stateObject.suicided || (deleteEmptyObjects && stateObject.empty()) {
				s.deleteStateObject(stateObject)
			} else {
				//调用update方法把存放在cache层的修改写入到trie数据库里面
				// 但是这个时候还没有写入底层的数据库。 还没有调用commit，数据还在内存里面，还没有落地成文件。
				stateObject.updateRoot(s.db)
				s.updateStateObject(stateObject)
			}
			// 将其加入修改过的对象缓存中
			s.stateObjectsDirty[addr] = struct{}{}
		}
		// Invalidate journal because reverting across transactions is not allowed.
		s.clearJournalAndRefund()
	}


updateStateObject将给定对象写入trie。

	func (self *StateDB) updateStateObject(stateObject *stateObject) {
		addr := stateObject.Address()
		// rlp编码
		data, err := rlp.EncodeToBytes(stateObject)
		if err != nil {
			panic(fmt.Errorf("can't encode object at %x: %v", addr[:], err))
		}
		// 调用TryUpdate更新trie中的数据
		// key 地址 value stateObject对象的分装
		self.setError(self.trie.TryUpdate(addr[:], data))
	}