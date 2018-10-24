# table

标签（空格分隔）： 未分类

---

table结构体
------
table是用于节点发现以及存储的重要结构。以下是table的结构体，和一些我个人的简单理解

    type Table struct {
    	mutex   sync.Mutex        // 锁用来保护 buckets桶结构, bucket content桶中的内容，也就是节点, nursery（种子节点）, rand随机数
    	buckets [nBuckets]*bucket // 按与本节点距离远近存放到桶中，这里设定为17个桶
    	nursery []*Node           // 启动节点，也就是初始同步时需要访问的节点
    	rand    *mrand.Rand       // 随即源，定期变换
    	ips     netutil.DistinctNetSet// DistinctNetSet数据结构描述为跟踪IP，保证最多n个属于同一网络？-todo
    
    	db         *nodeDB // 数据库中保存的已知节点
    	refreshReq chan chan struct{}  //刷新表的请求
    	initDone   chan struct{}       //初始化完成通道
    	closeReq   chan struct{}        //关闭请求通道
    	closed     chan struct{}        //关闭通道
    
    	bondmu    sync.Mutex        //与其他节点通信链接锁
    	bonding   map[NodeID]*bondproc //绑定的一个过程，用于验证节点的绑定状态
    	bondslots chan struct{} // 限制同时存在的绑定过程
    
    	nodeAddedHook func(*Node) // for testing
    
    	net  transport  //网络通信结构，包含Ping,查询节点等
    	self *Node // 自身节点
    }
    
以下是一些相关的内部结构体或者接口的定义

    // transport由UDP传输实现。 
    // 它是一个接口，因此我们可以在不打开大量UDP套接字且不生成私钥的情况下进行测试。
    // 暂时不理解上面这句话的意思-todo
    type transport interface {
    	ping(NodeID, *net.UDPAddr) error
    	waitping(NodeID) error
    	findnode(toid NodeID, addr *net.UDPAddr, target NodeID) ([]*Node, error)
    	close()
    }
    // ping 一个终端等待其回应，并更新相关的数据库
    func (tab *Table) ping(id NodeID, addr *net.UDPAddr) error {
    //更新当前最后一次ping的时间，用来判定超时，更新节点活跃列表等
    	tab.db.updateLastPing(id, time.Now())
    	//用net包中的ping方法，后续会在net包中详细描述
    	if err := tab.net.ping(id, addr); err != nil {
    		return err
    	}
    	//更新绑定时间
    	tab.db.updateBondTime(id, time.Now())
    	return nil
    }
    
    // 桶是存储节点的数据结构，按活跃度由前到后排序，最后的活跃度越低
    type bucket struct {
    	entries      []*Node //由最后一次链接时间排序
    	replacements []*Node //如果最近的验证失效则放入代替组?-todo
    	ips          netutil.DistinctNetSet 
    }
    
table的初始化过程包含了绝大多数的逻辑

newTable代码逻辑
--------------

> **<a href="#newTable">newTable(false)</a>**

>>**<a href="#setFallbackNodes">setFallbackNodes</a>** 设置启动链接节点

>>**<a href="#loadSeedNodes">loadSeedNodes(false)</a>**  加载种子节点

>>>**<a href="#add">add(seed)</a>**      将节点添加入桶中

>>>>**<a href="#bucket">bucket(new.sha)</a>**    返回节点对应的桶

>>>>**<a href="#bumpOrAdd">bumpOrAdd(b, new)</a>**将节点移动到bucket中节点列表的前面，或者如果列表未满，则添加它。 如果n在桶中，则返回值为true。

>>>>>**<a href="#pushNode">pushNode(b.entries, n, bucketSize)</a>**添加节点n，并置于首位，返回桶中节点列表和被删除的最后一位

>>**<a href="#ensureExpirer">ensureExpirer()</a>**数据库节点超时处理
>>**<a href="#loop">loop()</a>**维护table的核心逻辑

>>>**<a href="#doRefresh">doRefresh(refreshDone)</a>**执行随机目标的查找以保持桶满

>>>>**<a href="#bondall">bondall(seeds)</a>**将所有节点进行尝试性绑定，返回成功的节点

>>>>>**<a href="#bond">bond(false, n.ID, n.addr(), n.TCP)</a>**与该节点进行绑定

>>>>>>**<a href="#pingpong">pingpong</a>** pingpong交互函数

>>>>**<a href="#lookup">lookup(tab.self.ID, false)</a>**以自身节点ID为目标查找新的节点

>>>>>**<a href="#closest">closest(target, bucketSize)</a>**返回距离目标节点最近的16个节点

>>>>>>**<a href="#push">push(n, nresults)</a>**将给定节点加入列表，并使其小于最大容量限制

>>>**<a href="#doRevalidate">doRevalidate(revalidateDone)</a>**检查随机存储桶中的最后一个节点是否仍然存在，如果不存在，则替换或删除该节点。

>>>>**<a href="#nodeToRevalidate">nodeToRevalidate</a>**返回随机非空桶中的最后有一个节点



<a name="newTable">newTable</a>
----------


    func newTable(t transport, ourID NodeID, ourAddr *net.UDPAddr, nodeDBPath string, bootnodes []*Node) (*Table, error) {
    	// 创建或者链接已有数据库
    	db, err := newNodeDB(nodeDBPath, Version, ourID)
    	if err != nil {
    		return nil, err
    	}
    	tab := &Table{
    		net:        t,
    		db:         db,
    		self:       NewNode(ourID, ourAddr.IP, uint16(ourAddr.Port), uint16(ourAddr.Port)),
    		bonding:    make(map[NodeID]*bondproc),
    		bondslots:  make(chan struct{}, maxBondingPingPongs),//最多允许16个链接
    		refreshReq: make(chan chan struct{}),
    		initDone:   make(chan struct{}),
    		closeReq:   make(chan struct{}),
    		closed:     make(chan struct{}),
    		rand:       mrand.New(mrand.NewSource(0)),
    		ips:        netutil.DistinctNetSet{Subnet: tableSubnet, Limit: tableIPLimit},
    	}
    	if err := tab.setFallbackNodes(bootnodes); err != nil {
    		return nil, err
    	}
    	for i := 0; i < cap(tab.bondslots); i++ {
    		tab.bondslots <- struct{}{}
    	}
    	//初始化定义每个桶的ips
    	//但是不明白为什么table内已经初始化了，在桶内还有这个字段-todo
    	for i := range tab.buckets {
    		tab.buckets[i] = &bucket{
    			ips: netutil.DistinctNetSet{Subnet: bucketSubnet, Limit: bucketIPLimit},
    		}
    	}
    	//随机种子源
    	tab.seedRand()
    	//加载种子节点
    	tab.loadSeedNodes(false)
    	// Start the background expiration goroutine after loading seeds so that the search for
    	// seed nodes also considers older nodes that would otherwise be removed by the
    	// expiration.
    	//刷新数据库文件，删除超时节点
    	tab.db.ensureExpirer()
    	go tab.loop()
    	return tab, nil
    }




<a name="loadSeedNodes">loadSeedNodes</a>
----------
这里用到的loadSeedNodes，是用来加载种子节点。其中包含了节点入桶的add方法，如下。


    func (tab *Table) loadSeedNodes(bond bool) {
        //随机查找数据库中节点，作为种子节点
    	seeds := tab.db.querySeeds(seedCount, seedMaxAge)
    	//随机查找的种子再加上给定的初始链接节点
    	seeds = append(seeds, tab.nursery...)
    	//是否给所有节点绑定？
    	//目前不太明白这个布尔值的意义-todo
    	if bond {
    		seeds = tab.bondall(seeds)
    	}
    	for i := range seeds {
    		seed := seeds[i]
    		//返回最后一次pong成功到现在的时间
    		age := log.Lazy{Fn: func() interface{} { return time.Since(tab.db.bondTime(seed.ID)) }}
    		log.Debug("Found seed node in database", "id", seed.ID, "addr", seed.addr(), "age", age)
    		//add方法
    		tab.add(seed)
    	}
    }
<a name="add">add</a>
方法试图将节点加入桶中，如果桶中还有空间则立即加入，否则替换掉最近不响应ping的节点

    //调用者必须持有tab.mux 锁，因为需要改变桶中的内容
    func (tab *Table) add(new *Node) {
    	tab.mutex.Lock()
    	defer tab.mutex.Unlock()
        //bucket返回对应节点所在的桶
    	b := tab.bucket(new.sha)
    	//bumpOrAdd将节点移动到bucket中节点列表的前面，或者如果列表未满，则添加它。 如果n在桶中，则返回值为true。
    	if !tab.bumpOrAdd(b, new) {
    		// 如果节点不在桶内，或者桶中节点数量越界
    		//将节点添加到替换列表中
    		tab.addReplacement(b, new)
    	}
    }
        //返回节点所对应的桶
    func (tab *Table) bucket(sha common.Hash) *bucket {
    	//计算自身节点与该节点的距离
    	d := logdist(tab.self.sha, sha)
    	//如果小于最近的对数距离则放在桶0中
    	if d <= bucketMinDistance {
    		return tab.buckets[0]
    	}
    	return tab.buckets[d-bucketMinDistance-1]
    }
    func (tab *Table) bumpOrAdd(b *bucket, n *Node) bool {
    	if b.bump(n) {
    		return true
    	}
    	if len(b.entries) >= bucketSize || !tab.addIP(b, n.IP) {
    		return false
    	}
    	b.entries, _ = pushNode(b.entries, n, bucketSize)
    	//从replacements中删除n
    	b.replacements = deleteNode(b.replacements, n)
    	//更新节点进入的时间
    	n.addedAt = time.Now()
    	if tab.nodeAddedHook != nil {
    		tab.nodeAddedHook(n)
    	}
    	return true
    }
    // pushNode adds n to the front of list, keeping at most max items.
    
<a name="pushNode">pushNode</a>
如果桶满了删除桶中的最后一位

    func pushNode(list []*Node, n *Node, max int) ([]*Node, *Node) {
    	//这里为什么桶没满的时候添加的是nil-todo
    	//因为要将整个列表向后挪动一位（删除最后一位），并返回，所以当桶没满的时候就删除最后一位（空值）
    	if len(list) < max {
    		list = append(list, nil)
    	}
    	//删除最后一位，并返回它，将插入的节点赋值为第一位
    	removed := list[len(list)-1]
    	copy(list[1:], list)
    	list[0] = n
    	return list, removed
    }

<a name="ensureExpirer">ensureExpirer</a>
----------
种子节点加载完成后开始调用数据库的更新方法，主要用来节点的超时处理等，在[discover][1]中由详细描述ensureExpirer()。


  [1]: https://github.com/qdgogogo/go-ethereum-understanding/blob/master/discover.md
  
<a name="loop">loop</a>
----------
loop是实现维护整个table逻辑的核心函数

    func (tab *Table) loop() {
    	var (
    		revalidate     = time.NewTimer(tab.nextRevalidateTime())//10秒重新验证
    		refresh        = time.NewTicker(refreshInterval)//30分钟刷新
    		copyNodes      = time.NewTicker(copyNodesInterval)//30秒
    		revalidateDone = make(chan struct{})            //重验证完成通道
    		refreshDone    = make(chan struct{})           // 刷新完成通道
    		waiting        = []chan struct{}{tab.initDone} // 等待刷新通过到 runs
    	)
    	defer refresh.Stop()
    	defer revalidate.Stop()
    	defer copyNodes.Stop()
    
    	// 开始初始化的刷新过程
        //这里进行随机目标查找刷新表中的节点
    	go tab.doRefresh(refreshDone)
    //loop循环等待通道中的消息，以进行刷新或者关闭
    loop:
    	for {
    		select {
    		//30分钟刷新
    		case <-refresh.C:
    			//为什么要反复进行种子随机，保持随机性？
    			tab.seedRand()
    			if refreshDone == nil {
    				refreshDone = make(chan struct{})
    				go tab.doRefresh(refreshDone)
    			}
    		case req := <-tab.refreshReq:
    			//接收到refreshReq请求。那么进行刷新工作
    			waiting = append(waiting, req)
    			if refreshDone == nil {
    				refreshDone = make(chan struct{})
    				go tab.doRefresh(refreshDone)
    			}
    		case <-refreshDone:
    			//关闭每个等待的通道
    			for _, ch := range waiting {
    				close(ch)
    			}
    			waiting, refreshDone = nil, nil
    		case <-revalidate.C:
    			//10秒内随机重新验证
    			//这里是使用少量的资源保持桶的一定的”新鲜度“
    			go tab.doRevalidate(revalidateDone)
    		case <-revalidateDone:
    			//验证完后，初始化下一个验证的时间
    			revalidate.Reset(tab.nextRevalidateTime())
    		case <-copyNodes.C:
    			//30秒一次
    			//也就是将桶中的存在时间大于5分钟的节点加入数据库
    			go tab.copyBondedNodes()
    		case <-tab.closeReq:
    			break loop
    		}
    	}
    
    	if tab.net != nil {
    		tab.net.close()
    	}
    	if refreshDone != nil {
    		<-refreshDone
    	}
    	for _, ch := range waiting {
    		close(ch)
    	}
    	tab.db.close()
    	close(tab.closed)
    }
<a name="doRefresh">doRefresh</a>
执行随机目标的查找以保持桶满。如果表为空（初始引导程序或丢弃的故障对等程序），则插入种子节点。

    func (tab *Table) doRefresh(done chan struct{}) {
    	defer close(done)
    
    	// 从数据库加载节点并插入它们。 这应该会产生一些以前看到的节点仍然存在。
    	//这里的参数为true，所以逻辑会有不同
    	tab.loadSeedNodes(true)
    
    	//以自身节点ID为目标查找新的节点
    	//这里refreshIfEmpty参数为false
    	tab.lookup(tab.self.ID, false)
    

    	//Kademlia文件指定存储桶刷新应在最近最少使用的存储桶中执行查找。
    	// 我们不能坚持这一点，因为findnode目标是一个512位的值（不是散列大小），并且不容易生成属于所选存储桶的sha3前映像。 我们使用随机目标执行一些查找。
    	//以上是官方注释有待考究-todo
    	//进行三次随机目标查找
    	for i := 0; i < 3; i++ {
    		var target NodeID
    		crand.Read(target[:])
    		//随机字节填充目标ID进行随机查找
    		tab.lookup(target, false)
    	}
    }
<a name="bondall">bondall</a>
当loadSeedNodes(true)参数为true时，需要进行节点的绑定过程，也就是bondall函数

    //  bondall函数与所有给定节点同时绑定，并返回绑定可能已成功的节点。
    func (tab *Table) bondall(nodes []*Node) (result []*Node) {
    	rc := make(chan *Node, len(nodes))
    	//并发的绑定节点
    	for i := range nodes {
    		go func(n *Node) {
    			nn, _ := tab.bond(false, n.ID, n.addr(), n.TCP)
    			rc <- nn
    		}(nodes[i])
    	}
    	//并发的将节点加入到结果中
    	for range nodes {
    		if n := <-rc; n != nil {
    			result = append(result, n)
    		}
    	}
    	return result
    }
<a name="bond">bond</a>
确保本地节点与给定的远程节点具有绑定。如果绑定成功，它还会尝试将节点插入表中。在发送findnode请求之前必须建立绑定。双方必须完成ping/pong交换才能存在绑定。
活越的绑定过程的总数是有限的，以限制网络使用。
bond意味着在与远程节点的绑定中进行功能性操作，该远程节点仍然记住先前建立的绑定。 远程节点将不会发回ping，导致等待ping超时？这里暂时不理解-todo
如果pinged为true，则远程节点刚刚ping通我们，可以跳过一半的进程。

    func (tab *Table) bond(pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) (*Node, error) {
    	if id == tab.self.ID {
    		return nil, errors.New("is self")
    	}
    	//pinged为true并且初始化未完成
    	if pinged && !tab.isInitDone() {
    		return nil, errors.New("still initializing")
    	}
    	//如果我们有一段时间没有看到这个节点或者它在findnode经常失败，就开始绑定
    	node, fails := tab.db.node(id), tab.db.findFails(id)
    	//绑定了多长时间        
    	age := time.Since(tab.db.bondTime(id))
    	var result error
    	//如果失败次数大于0或者时间大于24小时则开始绑定过程
    	if fails > 0 || age > nodeDBNodeExpiration {
    		log.Trace("Starting bonding ping/pong", "id", id, "known", node != nil, "failcount", fails, "age", age)
    
    		tab.bondmu.Lock()
    		//w是bondproc，是绑定的过程
    		w := tab.bonding[id]
    		
    		if w != nil {
    			////如果该ID整备进行绑定，则等待绑定完成
    			tab.bondmu.Unlock()
    			<-w.done
    		} else {
    			// Register a new bonding process.
    			//注册一个新的绑定过程
    			w = &bondproc{done: make(chan struct{})}
    			tab.bonding[id] = w
    			tab.bondmu.Unlock()
    			// Do the ping/pong. The result goes into w.
    			tab.pingpong(w, pinged, id, addr, tcpPort)
    			// Unregister the process after it's done.
    			tab.bondmu.Lock()
    			//删除这个ID的处理过程
    			delete(tab.bonding, id)
    			tab.bondmu.Unlock()
    		}
    		// 检索绑定结果
    		result = w.err
    		//node是刚才绑定成功的节点
    		if result == nil {
    			node = w.n
    		}
    	}
    	// 即使绑定ping / pong失败，也要将节点添加到表中。 如果它继续没有反应，它将很快被重新关联。
    	//如果他不为空，则添加该节点，并更新它的失败次数为0
    	if node != nil {
    		tab.add(node)
    		tab.db.updateFindFails(id, 0)
    	}
    	return node, result
    }
<a name="pingpong">pingpong</a>
pingpong函数逻辑实现

    func (tab *Table) pingpong(w *bondproc, pinged bool, id NodeID, addr *net.UDPAddr, tcpPort uint16) {
    	// 请求绑定槽以限制网络使用
    	<-tab.bondslots
    	defer func() { tab.bondslots <- struct{}{} }()
    
    	//Ping远程节点。并等待一个pong消息
    	if w.err = tab.ping(id, addr); w.err != nil {
    		close(w.done)
    		return
    	}
    	//这个在udp收到一个ping消息的时候被设置为真。这个时候我们已经收到对方的ping消息了。
    	if !pinged {
    		//在我们开始发送findnode请求之前，让远程节点有机会ping我们。 如果他们还记得我们，那么waitping只会超时。
    		//这里不太理解-todo
    		tab.net.waitping(id)
    	}
    	// Bonding succeeded, update the node database.
    	//完成bond过程。
    	w.n = NewNode(id, addr.IP, uint16(addr.Port), tcpPort)
    	close(w.done)
    }


<a name="lookup">lookup</a>
这个函数用来查询一个指定节点的信息。 这个函数首先从本地拿到距离这个节点最近的所有16个节点。然后给所有的节点发送findnode的请求。 然后对返回的节点进行bondall处理, 然后返回所有的节点。

    func (tab *Table) lookup(targetID NodeID, refreshIfEmpty bool) []*Node {
    	var (
    		target         = crypto.Keccak256Hash(targetID[:])
    		asked          = make(map[NodeID]bool)
    		seen           = make(map[NodeID]bool)
    		reply          = make(chan []*Node, alpha)
    		pendingQueries = 0
    		result         *nodesByDistance
    	)
    	// 将自己标记为已查
    	asked[tab.self.ID] = true
    
    	for {
    		//每次加锁意味着会有改变table的操作
    		tab.mutex.Lock()
    		
    		//返回按距离目标节点远近，由小到大排列的集合
    		//求取和target最近的16个节点
    		result = tab.closest(target, bucketSize)
    		tab.mutex.Unlock()
    		//在doRefresh中refreshIfEmpty参数为false再次跳出，只有在被调用Lookup函数时为true
    		//在初始化table时会跳过主动刷新过程
    		if len(result.entries) > 0 || !refreshIfEmpty {
    			break
    		}
    		
    		//列表为空。我们实际上等待刷新完成。 第一个查询将遇到这种情况并运行引导逻辑。
    		//主动请求刷新
    		
    		<-tab.refresh()
    		refreshIfEmpty = false
    	}
    
    	for {
    		// 询问我们尚未访问的最接近的节点
    		//alpha限制最多同时进行3个查找过程？
    		// 每次迭代会查询result中和target距离最近的三个节点。
    		for i := 0; i < len(result.entries) && pendingQueries < alpha; i++ {
    			n := result.entries[i]
    			if !asked[n.ID] {
    				asked[n.ID] = true
    				pendingQueries++
    				go func() {

    					//向该节点发送查找节点
    					//r中包含了查找结果（距离目标最近的16个节点）
    					r, err := tab.net.findnode(n.ID, n.addr(), targetID)
    					if err != nil {
    						//失败次数增加
    						fails := tab.db.findFails(n.ID) + 1
    						tab.db.updateFindFails(n.ID, fails)
    						log.Trace("Bumping findnode failure counter", "id", n.ID, "failcount", fails)
    						//如果失败次数过多就删除该节点
    						if fails >= maxFindnodeFailures {
    							log.Trace("Too many findnode failures, dropping", "id", n.ID, "failcount", fails)
    							tab.delete(n)
    						}
    					}
    					//给这些节点发送绑定请求，返回结果
    					//reply最多缓存3组结果
    					reply <- tab.bondall(r)
    				}()
    			}
    		}
    		if pendingQueries == 0 {
    			//已经查找完最近的所有节点
    			break
    		}
    		//继续将结果放入桶中并按距离排序
    		for _, n := range <-reply {
    			//因为不同的远方节点可能返回相同的节点。所有用seen[]来做排重。
    			if n != nil && !seen[n.ID] {
    				seen[n.ID] = true
    				//找出来的结果又会加入result这个队列。也就是说这是一个循环查找的过程， 只要result里面不断加入新的节点。这个循环就不会终止。
    				result.push(n, bucketSize)
    			}
    		}
    		pendingQueries--
    	}
    	return result.entries
    }
    
closest返回具体目标节点最近的id

    func (tab *Table) closest(target common.Hash, nresults int) *nodesByDistance {

    	//这是找到最近节点但非常正确的非常浪费的方法。 我相信基于树的桶可以使这更容易有效地实现。
    	//以上时官方注释，基于树的桶有待研究-todo
    	close := &nodesByDistance{target: target}
    	for _, b := range tab.buckets {
    		for _, n := range b.entries {
    			close.push(n, nresults)
    		}
    	}
    	return close
    }
<a name="push">push</a>push
将给定节点加入列表，并使其小于最大容量限制

    func (h *nodesByDistance) push(n *Node, maxElems int) {
    	//采用二分法搜索找到[0, n)区间内最小的满足f(i)==true的值i
    	//也就是说该函数为true时n节点距离目标更近
    	ix := sort.Search(len(h.entries), func(i int) bool {
    		return distcmp(h.target, h.entries[i].sha, n.sha) > 0
    	})
    	//如果桶没满就添加到h中
    	if len(h.entries) < maxElems {
    		h.entries = append(h.entries, n)
    	}
    	if ix == len(h.entries) {
    		// farther away than all nodes we already have.比我们已经拥有的所有节点都远。也就是找到的最近索引位置比现有的都远。
    		// if there was room for it, the node is now the last element.如果有空间，节点现在是最后一个元素。
    	} else {
    		// slide existing entries down to make room
    		// this will overwrite the entry we just appended.
    		//向下滑动现有条目以腾出空间，这将覆盖我们刚刚附加的条目。也就是按距离顺序排好，由近到远。
    		copy(h.entries[ix+1:], h.entries[ix:])
    		h.entries[ix] = n
    	}
    }
<a name="doRevalidate">doRevalidate</a>
检查随机存储桶中的最后一个节点是否仍然存在，如果不存在，则替换或删除该节点。

    func (tab *Table) doRevalidate(done chan<- struct{}) {
    	defer func() { done <- struct{}{} }()
    
    	last, bi := tab.nodeToRevalidate()
    	if last == nil {
    		// No non-empty bucket found.
    		return
    	}
    
    	// Ping the selected node and wait for a pong.
    	err := tab.ping(last.ID, last.addr())
    
    	tab.mutex.Lock()
    	defer tab.mutex.Unlock()
    	b := tab.buckets[bi]
    	if err == nil {
    		// The node responded, move it to the front.
    		log.Debug("Revalidated node", "b", bi, "id", last.ID)
    		//移动到第一位
    		b.bump(last)
    		return
    	}
    	// No reply received, pick a replacement or delete the node if there aren't
    	// any replacements.
    	//替换或者删除该节点
    	if r := tab.replace(b, last); r != nil {
    		log.Debug("Replaced dead node", "b", bi, "id", last.ID, "ip", last.IP, "r", r.ID, "rip", r.IP)
    	} else {
    		log.Debug("Removed dead node", "b", bi, "id", last.ID, "ip", last.IP)
    	}
    }
    
<a name="nodeToRevalidate">nodeToRevalidate</a>
返回随机非空桶中的最后一个节点

    func (tab *Table) nodeToRevalidate() (n *Node, bi int) {
    	tab.mutex.Lock()
    	defer tab.mutex.Unlock()
    	//随机一个桶
    	for _, bi = range tab.rand.Perm(len(tab.buckets)) {
    		b := tab.buckets[bi]
    		if len(b.entries) > 0 {
    			last := b.entries[len(b.entries)-1]
    			return last, bi
    		}
    	}
    	return nil, 0
    }

对外暴露的方法
-------
ReadRandomNodes
ReadRandomNodes使用表中的随机节点填充给定切片。 它不会多次写入同一节点。 切片中的节点是副本，可以由调用者修改。

    func (tab *Table) ReadRandomNodes(buf []*Node) (n int) {
    	if !tab.isInitDone() {
    		return 0
    	}
    	tab.mutex.Lock()
    	defer tab.mutex.Unlock()
    
    	//到所有非空桶并获得新的切片。
    	var buckets [][]*Node
    	for _, b := range tab.buckets {
    		if len(b.entries) > 0 {
    			buckets = append(buckets, b.entries[:])
    		}
    	}
    	if len(buckets) == 0 {
    		return 0
    	}
    	//随机混乱桶,类似于洗牌
    	for i := len(buckets) - 1; i > 0; i-- {
    		j := tab.rand.Intn(len(buckets))
    		buckets[i], buckets[j] = buckets[j], buckets[i]
    	}
    	// Move head of each bucket into buf, removing buckets that become empty.
    	//将每个桶的头部移动到buf中,让buckets变空
    	var i, j int
    	for ; i < len(buf); i, j = i+1, (j+1)%len(buckets) {
    		b := buckets[j]
    		//将桶中的第一个节点移动到buf[i]
    		buf[i] = &(*b[0])
    		buckets[j] = b[1:]
    		if len(b) == 1 {
    			buckets = append(buckets[:j], buckets[j+1:]...)
    		}
    		if len(buckets) == 0 {
    			break
    		}
    	}
    	return i + 1
    }


Resolve 
用来获取一个指定ID的节点,如果节点在本地。那么返回本地节点。否则执行Lookup在网络上查询一次。 如果查询到节点。那么返回。否则返回nil

    func (tab *Table) Resolve(targetID NodeID) *Node {
    	// 如果节点存在于本地表中，则不需要网络交互。
    	hash := crypto.Keccak256Hash(targetID[:])
    	tab.mutex.Lock()
    	cl := tab.closest(hash, 1)
    	tab.mutex.Unlock()
    	if len(cl.entries) > 0 && cl.entries[0].ID == targetID {
    		return cl.entries[0]
    	}
    	// 使用lookup查询
    	result := tab.Lookup(targetID)
    	for _, n := range result {
    		if n.ID == targetID {
    			return n
    		}
    	}
    	return nil
    }
<a name="setFallbackNodes">setFallbackNodes</a>
如果表为空并且数据库中没有已知节点，则这些节点用于连接到网络

    func (tab *Table) setFallbackNodes(nodes []*Node) error {
    	for _, n := range nodes {
    		//验证节点是否有效
    		if err := n.validateComplete(); err != nil {
    			return fmt.Errorf("bad bootstrap/fallback node %q (%v)", n, err)
    		}
    	}
    	tab.nursery = make([]*Node, 0, len(nodes))
    	for _, n := range nodes {
    		cpy := *n
    		// Recompute cpy.sha because the node might not have been
    		// created by NewNode or ParseNode.
    		//重新计算cpy.sha，因为该节点可能尚未由NewNode或ParseNode创建。？？-todo
    		cpy.sha = crypto.Keccak256Hash(n.ID[:])
    		//将根节点加入初始联系节点
    		tab.nursery = append(tab.nursery, &cpy)
    	}
    	return nil
    }

