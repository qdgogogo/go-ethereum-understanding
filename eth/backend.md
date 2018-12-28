# backend #

----------
以太坊全节点服务

	type Ethereum struct {
		config      *Config //以太坊服务配置
		chainConfig *params.ChainConfig//链配置
	
		// Channel for shutting down the service
		shutdownChan chan bool // Channel for shutting down the Ethereum
	
		// Handlers
		txPool          *core.TxPool	//交易池
		blockchain      *core.BlockChain//区块链
		protocolManager *ProtocolManager//协议管理
		lesServer       LesServer	//轻量级客户端服务
	
		// DB interfaces
		// 数据库接口
		chainDb ethdb.Database // Block chain database
	
		eventMux       *event.TypeMux//
		engine         consensus.Engine//共识引擎
		accountManager *accounts.Manager//账号管理
	
		// 接收bloom过滤器数据请求的通道
		bloomRequests chan chan *bloombits.Retrieval // Channel receiving bloom data retrieval requests
		// 在区块import的时候执行Bloom indexer操作
		bloomIndexer  *core.ChainIndexer             // Bloom indexer operating during block imports
	
		APIBackend *EthAPIBackend	//提供给RPC服务使用的API后端
	
		miner     *miner.Miner	//矿工
		gasPrice  *big.Int	//节点接收的gasPrice的最小值。 比这个值更小的交易会被本节点拒绝
		etherbase common.Address //矿工地址
	
		networkID     uint64	//网络ID  testnet是0 mainnet是1
		netRPCService *ethapi.PublicNetAPI //RPC的服务
	
		lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)
	}


New创建一个新的eth对象

	func New(ctx *node.ServiceContext, config *Config) (*Ethereum, error) {
		// Ensure configuration values are compatible and sane
		// 确保配置值兼容且清晰
	
		//同步模式
		if config.SyncMode == downloader.LightSync {
			return nil, errors.New("can't run eth.Ethereum in light sync mode, use les.LightEthereum")
		}
		if !config.SyncMode.IsValid() {
			return nil, fmt.Errorf("invalid sync mode %d", config.SyncMode)
		}
		if config.MinerGasPrice == nil || config.MinerGasPrice.Cmp(common.Big0) <= 0 {
			log.Warn("Sanitizing invalid miner gas price", "provided", config.MinerGasPrice, "updated", DefaultConfig.MinerGasPrice)
			config.MinerGasPrice = new(big.Int).Set(DefaultConfig.MinerGasPrice)
		}
		// Assemble the Ethereum object
		// 组装以太坊对象
	
		//创建数据库
		chainDb, err := CreateDB(ctx, config, "chaindata")
		if err != nil {
			return nil, err
		}
		// 配置创世块
		chainConfig, genesisHash, genesisErr := core.SetupGenesisBlock(chainDb, config.Genesis)
		if _, ok := genesisErr.(*params.ConfigCompatError); genesisErr != nil && !ok {
			return nil, genesisErr
		}
		log.Info("Initialised chain configuration", "config", chainConfig)
	
		eth := &Ethereum{
			config:         config,
			chainDb:        chainDb,
			chainConfig:    chainConfig,
			eventMux:       ctx.EventMux,
			accountManager: ctx.AccountManager,
			engine:         CreateConsensusEngine(ctx, chainConfig, &config.Ethash, config.MinerNotify, config.MinerNoverify, chainDb),
			shutdownChan:   make(chan bool),
			networkID:      config.NetworkId,// 网络ID用来区别网路。 测试网络是0.主网是1
			gasPrice:       config.MinerGasPrice,
			etherbase:      config.Etherbase,
			bloomRequests:  make(chan chan *bloombits.Retrieval),
			bloomIndexer:   NewBloomIndexer(chainDb, params.BloomBitsBlocks, params.BloomConfirms),
		}
	
		log.Info("Initialising Ethereum protocol", "versions", ProtocolVersions, "network", config.NetworkId)
	
		// 检查版本
		if !config.SkipBcVersionCheck {
			bcVersion := rawdb.ReadDatabaseVersion(chainDb)
			if bcVersion != core.BlockChainVersion && bcVersion != 0 {
				return nil, fmt.Errorf("Blockchain DB version mismatch (%d / %d).\n", bcVersion, core.BlockChainVersion)
			}
			rawdb.WriteDatabaseVersion(chainDb, core.BlockChainVersion)
		}
		var (
			vmConfig = vm.Config{
				EnablePreimageRecording: config.EnablePreimageRecording,
				EWASMInterpreter:        config.EWASMInterpreter,
				EVMInterpreter:          config.EVMInterpreter,
			}
			cacheConfig = &core.CacheConfig{Disabled: config.NoPruning, TrieNodeLimit: config.TrieCache, TrieTimeLimit: config.TrieTimeout}
		)
		// 创建新的区块链对象
		eth.blockchain, err = core.NewBlockChain(chainDb, cacheConfig, eth.chainConfig, eth.engine, vmConfig, eth.shouldPreserve)
		if err != nil {
			return nil, err
		}
		// Rewind the chain in case of an incompatible config upgrade.
		// 如果配置升级不兼容，回滚链。
		if compat, ok := genesisErr.(*params.ConfigCompatError); ok {
			log.Warn("Rewinding chain to upgrade configuration", "err", compat)
			eth.blockchain.SetHead(compat.RewindTo)
			rawdb.WriteChainConfig(chainDb, genesisHash, chainConfig)
		}
		// 链索引
		eth.bloomIndexer.Start(eth.blockchain)
	
		if config.TxPool.Journal != "" {
			config.TxPool.Journal = ctx.ResolvePath(config.TxPool.Journal)
		}
		// 创建交易池。 用来存储本地或者在网络上接收到的交易。
		eth.txPool = core.NewTxPool(config.TxPool, eth.chainConfig, eth.blockchain)
	
		// 创建协议管理器
		if eth.protocolManager, err = NewProtocolManager(eth.chainConfig, config.SyncMode, config.NetworkId, eth.eventMux, eth.txPool, eth.engine, eth.blockchain, chainDb); err != nil {
			return nil, err
		}
		// 创建矿工
		eth.miner = miner.New(eth, eth.chainConfig, eth.EventMux(), eth.engine, config.MinerRecommit, config.MinerGasFloor, config.MinerGasCeil, eth.isLocalBlock)
		eth.miner.SetExtra(makeExtraData(config.MinerExtraData))
		// 用于给RPC调用提供后端支持
		eth.APIBackend = &EthAPIBackend{eth, nil}
		gpoParams := config.GPO
		if gpoParams.Default == nil {
			gpoParams.Default = config.MinerGasPrice
		}
		// gasprice 预测。通过最近的交易来预测当前的GasPrice的值。这个值可以作为之后发送交易的费用的参考
		eth.APIBackend.gpo = gasprice.NewOracle(eth.APIBackend, gpoParams)
	
		return eth, nil
	}

创建链的数据库

	func CreateDB(ctx *node.ServiceContext, config *Config, name string) (ethdb.Database, error) {
		db, err := ctx.OpenDatabase(name, config.DatabaseCache, config.DatabaseHandles)
		if err != nil {
			return nil, err
		}
		if db, ok := db.(*ethdb.LDBDatabase); ok {
			db.Meter("eth/db/chaindata/")
		}
		return db, nil
	}


Start实现node.Service，启动以太坊协议实现所需的所有内部goroutine。

在node.start()中会启动每个服务，包括以太坊服务

	func (s *Ethereum) Start(srvr *p2p.Server) error {
		// Start the bloom bits servicing goroutines
		// 启动bloom服务goroutines
		s.startBloomHandlers(params.BloomBitsBlocks)
	
		// Start the RPC service
		// 启动RPC服务
		s.netRPCService = ethapi.NewPublicNetAPI(srvr, s.NetVersion())
	
		// Figure out a max peers count based on the server limits
		// 根据服务器限制计算出最大peer计数
		maxPeers := srvr.MaxPeers
		if s.config.LightServ > 0 {
			if s.config.LightPeers >= srvr.MaxPeers {
				return fmt.Errorf("invalid peer config: light peer count (%d) >= total peer count (%d)", s.config.LightPeers, srvr.MaxPeers)
			}
			maxPeers -= s.config.LightPeers
		}
		// Start the networking layer and the light server if requested
		// 启动网络层，如果需要启动轻服务
		s.protocolManager.Start(maxPeers)
		if s.lesServer != nil {
			s.lesServer.Start(srvr)
		}
		return nil
	}