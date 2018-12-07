# state_object #

----------
state_object是account对象的抽象，提供了账户的一些功能

stateObject表示正在修改的以太坊帐户。

使用模式如下：首先，您需要获取状态对象。 可以通过对象访问和修改帐户值。 最后，调用CommitTrie将修改后的存储trie写入数据库。
	
	type stateObject struct {
		address  common.Address //地址
		// 地址的哈希值
		addrHash common.Hash // hash of ethereum address of the account
		// 账户
		data     Account
		db       *StateDB //数据库
	
		// DB error.
		// State objects are used by the consensus core and VM which are
		// unable to deal with database-level errors. Any error that occurs
		// during a database read is memoized here and will eventually be returned
		// by StateDB.Commit.
		dbErr error
	
		// Write caches.
		// 写缓存
		trie Trie // 用户的存储trie ，在第一次访问的时候变得非空
		code Code //  合约代码，当代码被加载的时候被设置
	
		originStorage Storage // 缓存原始条目以进行重复数据删除重写
		dirtyStorage  Storage // 需要存到到磁盘的存储实体
	
		// Cache flags.
		// When an object is marked suicided it will be delete from the trie
		// during the "update" phase of the state transition.
		// 状态转换
		// 当一个对象被标记为自杀时，它将在状态转换的“更新”阶段从trie中删除。
		dirtyCode bool // true if the code was updated
		suicided  bool
		deleted   bool
	}

Account账户是以太币的账户共识表示。 这些对象存储在主帐户trie中。
	
	type Account struct {
		Nonce    uint64
		Balance  *big.Int
		Root     common.Hash // merkle root of the storage trie
		CodeHash []byte
	}


UpdateRoot将trie root设置为当前的根哈希值

	func (self *stateObject) updateRoot(db Database) {
		self.updateTrie(db)
		self.data.Root = self.trie.Hash()
	}

updateTrie将缓存存储修改写入对象的存储trie。
将dirtyStorage中的修改更新到originStorage上。

	func (self *stateObject) updateTrie(db Database) Trie {
		// 取得该对象的Trie
		tr := self.getTrie(db)
		// 遍历将修改应用
		for key, value := range self.dirtyStorage {
			delete(self.dirtyStorage, key)
	
			// Skip noop changes, persist actual changes
			// 跳过无用的变化，实例化真正的修改，也就是说和原来一样就跳过，如果改变就更新
			if value == self.originStorage[key] {
				continue
			}
			self.originStorage[key] = value
	
			if (value == common.Hash{}) {
				self.setError(tr.TryDelete(key[:]))
				continue
			}
			// Encoding []byte cannot fail, ok to ignore the error.
			v, _ := rlp.EncodeToBytes(bytes.TrimLeft(value[:], "\x00"))
			self.setError(tr.TryUpdate(key[:], v))
		}
		return tr
	}