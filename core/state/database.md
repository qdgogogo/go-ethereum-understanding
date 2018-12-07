# database #

----------
Database包含树和合同代码的访问。

	type Database interface {
		// OpenTrie opens the main account trie.
		// OpenTrie 打开了主账号的trie树
		OpenTrie(root common.Hash) (Trie, error)
	
		// OpenStorageTrie opens the storage trie of an account.
		// OpenStorageTrie 打开了一个账号的storage trie
		OpenStorageTrie(addrHash, root common.Hash) (Trie, error)
		
		// CopyTrie returns an independent copy of the given trie.
		// 返回给定trie的独立副本。
		CopyTrie(Trie) Trie
	
		// ContractCode retrieves a particular contract's code.
		// 检索特定合同的代码。
		ContractCode(addrHash, codeHash common.Hash) ([]byte, error)
	
		// ContractCodeSize retrieves a particular contracts code's size.
		// 返回给定合约的代码大小
		ContractCodeSize(addrHash, codeHash common.Hash) (int, error)
	
		// TrieDB retrieves the low level trie database used for data storage.
		// 返回底层用于存储数据的数据库
		TrieDB() *trie.Database
	}

Trie是以太坊Merkle Trie
	
	type Trie interface {
		TryGet(key []byte) ([]byte, error)
		TryUpdate(key, value []byte) error
		TryDelete(key []byte) error
		Commit(onleaf trie.LeafCallback) (common.Hash, error)
		Hash() common.Hash
		NodeIterator(startKey []byte) trie.NodeIterator
		GetKey([]byte) []byte // TODO(fjl): remove this when SecureTrie is removed
		Prove(key []byte, fromLevel uint, proofDb ethdb.Putter) error
	}


打开主账户树

	func (db *cachingDB) OpenTrie(root common.Hash) (Trie, error) {
		db.mu.Lock()
		defer db.mu.Unlock()
	
		for i := len(db.pastTries) - 1; i >= 0; i-- {
			if db.pastTries[i].Hash() == root {
				return cachedTrie{db.pastTries[i].Copy(), db}, nil
			}
		}
		tr, err := trie.NewSecure(root, db.db, MaxTrieCacheGen)
		if err != nil {
			return nil, err
		}
		return cachedTrie{tr, db}, nil
	}
