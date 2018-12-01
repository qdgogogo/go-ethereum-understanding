# 以太坊数据持久化

标签（空格分隔）： ethdb

---

ethdb包
------


----------


interface.go
------------

定义了数据库的基本操作

    // 经验性的定义了批处理的大小？
    const IdealBatchSize = 100 * 1024
    // Putter 是写操作的接口，普通写入和批处理写入都要用到。
    type Putter interface {
    	Put(key []byte, value []byte) error
    }
    // 数据库的基本操作，写入，获取，判断存在，删除，关闭，新的批处理
    type Database interface {
    	Putter
    	Get(key []byte) ([]byte, error)
    	Has(key []byte) (bool, error)
    	Delete(key []byte) error
    	Close()
    	NewBatch() Batch
    }

    //Batch是一个只写的接口。不能多线程同时使用。
    
    type Batch interface {
    	Putter
    	ValueSize() int // 批处理的数量
    	Write() error
    	// Reset 重置批处理
    	Reset()
    }
database.go
------------
   

    type LDBDatabase struct {
	fn string      // 文件名
	db *leveldb.DB // LevelDB 实例
    //metrics测量与监控
	getTimer       metrics.Timer 
	putTimer       metrics.Timer 
	delTimer       metrics.Timer 
	missMeter      metrics.Meter 
	readMeter      metrics.Meter 
	writeMeter     metrics.Meter 
	compTimeMeter  metrics.Meter 
	compReadMeter  metrics.Meter 
	compWriteMeter metrics.Meter 

	quitLock sync.Mutex      // 互斥锁保护退出通道访问
	quitChan chan chan error // 在关闭数据库之前退出通道以停止度量标准收集

	log log.Logger 
    }

    // 返回一个LDB数据库对象
    func NewLDBDatabase(file string, cache int, handles int) (*LDBDatabase, error) {
    	logger := log.New("database", file)
    
    //确保最小的缓存和文件保证？-todo
    if cache < 16 {
    	cache = 16
    }
    if handles < 16 {
    	handles = 16
    }
    logger.Info("Allocated cache and file handles", "cache", cache, "handles", handles)
    
    // 打开数据库并恢复任何潜在的损坏
    db, err := leveldb.OpenFile(file, &opt.Options{
    	OpenFilesCacheCapacity: handles, //定义打开文件缓存的容量。
    	BlockCacheCapacity:     cache / 2 * opt.MiB,  //在levedb中该项是'sorted table'块缓存容量
    	WriteBuffer:            cache / 4 * opt.MiB,  // 其中的两个在内部使用？-todo
    	Filter:                 filter.NewBloomFilter(10),//指定过滤器
    })
    if _, corrupted := err.(*errors.ErrCorrupted); corrupted {
    	db, err = leveldb.RecoverFile(file, nil)
    }
    // (Re)check for errors and abort if opening of the db failed
    if err != nil {
    	return nil, err
    }
    return &LDBDatabase{
    		fn:  file,
    		db:  db,
    		log: logger,
    	}, nil
    }

put方法，之后的方法与put类似

    func (db *LDBDatabase) Put(key []byte, value []byte) error {
    	// 用来测量put的延迟
    	if db.putTimer != nil {
    		defer db.putTimer.UpdateSince(time.Now())
    	}
    	// Generate the data to write to disk, update the meter and write
    	//这个不知道是什么意思？-todo
    	//value = rle.Compress(value)
        //标记
    	if db.writeMeter != nil {
    		db.writeMeter.Mark(int64(len(value)))
    	}
    	//调用levedb的put
    	return db.db.Put(key, value, nil)
    }
Metrics的处理

    func (db *LDBDatabase) Meter(prefix string) {
    	// 判断是否禁用测量
    	if !metrics.Enabled {
    		return
    	}
    	// 初始化所有度量收集器
    	db.getTimer = metrics.NewRegisteredTimer(prefix+"user/gets", nil)
    	db.putTimer = metrics.NewRegisteredTimer(prefix+"user/puts", nil)
    	db.delTimer = metrics.NewRegisteredTimer(prefix+"user/dels", nil)
    	db.missMeter = metrics.NewRegisteredMeter(prefix+"user/misses", nil)
    	db.readMeter = metrics.NewRegisteredMeter(prefix+"user/reads", nil)
    	db.writeMeter = metrics.NewRegisteredMeter(prefix+"user/writes", nil)
    	db.compTimeMeter = metrics.NewRegisteredMeter(prefix+"compact/time", nil)
    	db.compReadMeter = metrics.NewRegisteredMeter(prefix+"compact/input", nil)
    	db.compWriteMeter = metrics.NewRegisteredMeter(prefix+"compact/output", nil)
    
    	// 为定期收集器创建退出通道并运行它
    	db.quitLock.Lock()
    	db.quitChan = make(chan chan error)
    	db.quitLock.Unlock()
        //调用下面的meter方法
    	go db.meter(3 * time.Second)
    }
这个方法每3秒钟获取一次leveldb内部的计数器，然后把他们公布到metrics子系统。这是一个无限循环的方法， 直到quitChan收到了一个退出信号。

    func (db *LDBDatabase) meter(refresh time.Duration) {
    	// 创建[2][3]大小的数组存储数据
    	counters := make([][]float64, 2)
    	for i := 0; i < 2; i++ {
    		counters[i] = make([]float64, 3)
    	}
    	for i := 1; ; i++ {
    		// 获取数据库状态
    		stats, err := db.db.GetProperty("leveldb.stats")
    		if err != nil {
    			db.log.Error("Failed to read database stats", "err", err)
    			return
    		}
    		// 找到对应的表
    		lines := strings.Split(stats, "\n")
    		for len(lines) > 0 && strings.TrimSpace(lines[0]) != "Compactions" {
    			lines = lines[1:]
    		}
    		if len(lines) <= 3 {
    			db.log.Error("Compaction table not found")
    			return
    		}
    		lines = lines[3:]
    
    		// 迭代行，获取需要的数据
    		for j := 0; j < len(counters[i%2]); j++ {
    			counters[i%2][j] = 0
    		}
    		for _, line := range lines {
    			parts := strings.Split(line, "|")
    			if len(parts) != 6 {
    				break
    			}
    			for idx, counter := range parts[3:] {
    				value, err := strconv.ParseFloat(strings.TrimSpace(counter), 64)
    				if err != nil {
    					db.log.Error("Compaction entry parsing failed", "err", err)
    					return
    				}
    				counters[i%2][idx] += value
    			}
    		}
    		// 更新所需要的指标
    		if db.compTimeMeter != nil {
    			db.compTimeMeter.Mark(int64((counters[i%2][0] - counters[(i-1)%2][0]) * 1000 * 1000 * 1000))
    		}
    		if db.compReadMeter != nil {
    			db.compReadMeter.Mark(int64((counters[i%2][1] - counters[(i-1)%2][1]) * 1024 * 1024))
    		}
    		if db.compWriteMeter != nil {
    			db.compWriteMeter.Mark(int64((counters[i%2][2] - counters[(i-1)%2][2]) * 1024 * 1024))
    		}
    		// 使用select处理信息
    		select {
    		case errc := <-db.quitChan:
    			errc <- nil
    			return
    
    		case <-time.After(refresh):
    		}
    	}
    }

