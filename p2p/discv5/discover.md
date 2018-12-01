# discover

标签（空格分隔）： 未分类

---

database.go
-----------
实现了节点的持久化，例如新建数据文件，删除过期节点等。
结构体定义如下：

    // 用来存储我们所知道的所有节点
    type nodeDB struct {
    	lvl    *leveldb.DB   // 数据库接口
    	self   NodeID        // 自身的id，不用加入到数据库中
    	runner sync.Once     // 用来保证只启动一次
    	quit   chan struct{} // 推出的通道
    }
根据所给的参数，来确定创建的是内存数据库还是文件数据库

    // 根据路径参数确定，如果没给定路径则使用内存数据库
    func newNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
    	if path == "" {
    		return newMemoryNodeDB(self)
    	}
    	return newPersistentNodeDB(path, version, self)
    }
以下主要说明基于文件的数据库

    func newPersistentNodeDB(path string, version int, self NodeID) (*nodeDB, error) {
    	opts := &opt.Options{OpenFilesCacheCapacity: 5}
    	db, err := leveldb.OpenFile(path, opts) //创建或者打开levedb
    	if _, iscorrupted := err.(*errors.ErrCorrupted); iscorrupted {
    		db, err = leveldb.RecoverFile(path, nil)
    	}
    	if err != nil {
    		return nil, err
    	}
    	// 获取缓存中节点的版本，如果不匹配则刷新
    	currentVer := make([]byte, binary.MaxVarintLen64)
    	currentVer = currentVer[:binary.PutVarint(currentVer, int64(version))]
    
    	blob, err := db.Get(nodeDBVersionKey, nil)
    	switch err {
    	case leveldb.ErrNotFound:
    		// 如果找不到版本则把当前的版本值插入
    		if err := db.Put(nodeDBVersionKey, currentVer, nil); err != nil {
    			db.Close()
    			return nil, err
    		}
    
    	case nil:
    		// 更新版本
    		if !bytes.Equal(blob, currentVer) {
    			db.Close()
    			//将数据库文件删除重新创建
    			if err = os.RemoveAll(path); err != nil {
    				return nil, err
    			}
    			return newPersistentNodeDB(path, version, self)
    		}
    	}
    	return &nodeDB{
    		lvl:  db,
    		self: self,
    		quit: make(chan struct{}),
    	}, nil
    }
查找和更新操作

    // 返回给定id的node
    func (db *nodeDB) node(id NodeID) *Node {
    	blob, err := db.lvl.Get(makeKey(id, nodeDBDiscoverRoot), nil)
    	if err != nil {
    		return nil
    	}
    	node := new(Node)
    	if err := rlp.DecodeBytes(blob, node); err != nil {
    		log.Error("Failed to decode node RLP", "err", err)
    		return nil
    	}
    	node.sha = crypto.Keccak256Hash(node.ID[:])
    	return node
    }
    
    // 更新对应id的Node值
    func (db *nodeDB) updateNode(node *Node) error {
    	blob, err := rlp.EncodeToBytes(node)
    	if err != nil {
    		return err
    	}
    	return db.lvl.Put(makeKey(node.ID, nodeDBDiscoverRoot), blob, nil)
    }

节点的超时处理
  

     //只运行一次
     //这个方法设置的目的是为了在网络成功启动后在开始进行数据超时丢弃的工作(以防一些潜在的有用的种子节点被丢弃)？--todo 需要整体阅读后才做理解。 
        func (db *nodeDB) ensureExpirer() {
        	db.runner.Do(func() { go db.expirer() })
        }
    
    // 每隔一个小时检查一次？
    func (db *nodeDB) expirer() {
    	tick := time.NewTicker(nodeDBCleanupCycle)
    	defer tick.Stop()
    	for {
    		select {
    		case <-tick.C:
    			if err := db.expireNodes(); err != nil {
    				log.Error("Failed to expire nodedb items", "err", err)
    			}
    		case <-db.quit:
    			return
    		}
    	}
    }
    // 遍历数据库中的所有节点，一旦被判定超时则删除该节点，这样做的目的就是删除一些老旧的节点
    func (db *nodeDB) expireNodes() error {
    	threshold := time.Now().Add(-nodeDBNodeExpiration)//这的间隔是24小时
    
    	it := db.lvl.NewIterator(nil, nil)
    	defer it.Release()
    
    	for it.Next() {
    		// 跳过不是用发现字段标注的节点？-todo 还有什么类型的节点
    		id, field := splitKey(it.Key())
    		if field != nodeDBDiscoverRoot {
    			continue
    		}
    		// 跳过自身，跳过符合条件的节点
    		if !bytes.Equal(id[:], db.self[:]) {
    		    //也就是说从上一次成功的ping-pong超过24小时的节点就会被删除
    			if seen := db.bondTime(id); seen.After(threshold) {
    				continue
    			}
    		}
    		// Otherwise delete all associated information
    		db.deleteNode(id)
    	}
    	return nil
    }
从数据库中挑选合适的种子节点


    func (db *nodeDB) querySeeds(n int, maxAge time.Duration) []*Node {
    	var (
    		now   = time.Now()
    		nodes = make([]*Node, 0, n)
    		it    = db.lvl.NewIterator(nil, nil)
    		id    NodeID
    	)
    	defer it.Release()
    
    seek:
        //这里的5*n不知道根据什么定的
    	for seeks := 0; len(nodes) < n && seeks < n*5; seeks++ {
    		// 每次增加第一个字节随机量，以增加击中非常小的数据库中所有现有节点的可能性
    		ctr := id[0]
    		rand.Read(id[:])
    		id[0] = ctr + id[0]%16
    		it.Seek(makeKey(id, nodeDBDiscoverRoot))
    
    		n := nextNode(it)
    		if n == nil {
    			id[0] = 0
    			continue seek 
    		}
    		if n.ID == db.self {
    			continue seek
    		}
    		//从查找开始到该ID绑定时间的差值大于某个值
    		if now.Sub(db.bondTime(n.ID)) > maxAge {
    			continue seek
    		}
    		for i := range nodes {
    			if nodes[i].ID == n.ID {
    				continue seek // 如果是之前已经找到的那么继续？
    			}
    		}
    		nodes = append(nodes, n)
    	}
    	return nodes
    }

