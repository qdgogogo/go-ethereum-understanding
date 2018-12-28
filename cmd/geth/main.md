# main #

----------

如果没有运行特殊子命令，geth是进入系统的主要入口点。 它根据命令行参数创建一个默认节点，并以阻塞模式运行它，等待它关闭。

	func geth(ctx *cli.Context) error {
		if args := ctx.Args(); len(args) > 0 {
			return fmt.Errorf("invalid command: %q", args[0])
		}
		// 返回一个已经配置好的，一注册服务的节点
		node := makeFullNode(ctx)
		// 启动节点，服务
		startNode(ctx, node)
		// 阻塞以保持运行
		node.Wait()
		return nil
	}

startNode启动系统节点和所有已注册的协议，然后解锁所有请求的帐户，并启动RPC / IPC接口和矿工。

	func startNode(ctx *cli.Context, stack *node.Node) {
		debug.Memsize.Add("node", stack)
	
		// Start up the node itself
		// 启动节点
		utils.StartNode(stack)
	
		// Unlock any account specifically requested
		// 解锁任何特殊的帐户
		ks := stack.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)
	
		// 从全局的flag中读取密码文件
		passwords := utils.MakePasswordList(ctx)
		// 要解锁的账户
		unlocks := strings.Split(ctx.GlobalString(utils.UnlockedAccountFlag.Name), ",")
		for i, account := range unlocks {
			if trimmed := strings.TrimSpace(account); trimmed != "" {
				// 解锁对应的账户
				unlockAccount(ctx, ks, trimmed, i, passwords)
			}
		}
		// Register wallet event handlers to open and auto-derive wallets
		// 注册钱包事件处理程序以打开和自动派生钱包
		events := make(chan accounts.WalletEvent, 16)
		stack.AccountManager().Subscribe(events)
	
		// 开启新的协程
		go func() {
			// Create a chain state reader for self-derivation
			// 创建链状态读取器用来自我驱动
			rpcClient, err := stack.Attach()
			if err != nil {
				utils.Fatalf("Failed to attach to self: %v", err)
			}
			stateReader := ethclient.NewClient(rpcClient)
	
			// Open any wallets already attached
			// 打开已经关联的钱包
			for _, wallet := range stack.AccountManager().Wallets() {
				if err := wallet.Open(""); err != nil {
					log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
				}
			}
			// Listen for wallet event till termination
			// 监听钱包事件
			for event := range events {
				switch event.Kind {
				case accounts.WalletArrived:
					if err := event.Wallet.Open(""); err != nil {
						log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
					}
				case accounts.WalletOpened:
					status, _ := event.Wallet.Status()
					log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", status)
	
					derivationPath := accounts.DefaultBaseDerivationPath
					if event.Wallet.URL().Scheme == "ledger" {
						derivationPath = accounts.DefaultLedgerBaseDerivationPath
					}
					event.Wallet.SelfDerive(derivationPath, stateReader)
	
				case accounts.WalletDropped:
					log.Info("Old wallet dropped", "url", event.Wallet.URL())
					event.Wallet.Close()
				}
			}
		}()
		// Start auxiliary services if enabled
		// 如果启用则启动辅助服务
		if ctx.GlobalBool(utils.MiningEnabledFlag.Name) || ctx.GlobalBool(utils.DeveloperFlag.Name) {
			// Mining only makes sense if a full Ethereum node is running
			// 只有在运行完整的以太坊节点时才有意义挖掘
			if ctx.GlobalString(utils.SyncModeFlag.Name) == "light" {
				utils.Fatalf("Light clients do not support mining")
			}
			var ethereum *eth.Ethereum
			// 运行完整的以太坊服务才能挖矿
			if err := stack.Service(&ethereum); err != nil {
				utils.Fatalf("Ethereum service not running: %v", err)
			}
			// Set the gas price to the limits from the CLI and start mining
			// 从CLI设置gas价格开始挖掘
			gasprice := utils.GlobalBig(ctx, utils.MinerLegacyGasPriceFlag.Name)
			if ctx.IsSet(utils.MinerGasPriceFlag.Name) {
				gasprice = utils.GlobalBig(ctx, utils.MinerGasPriceFlag.Name)
			}
			ethereum.TxPool().SetGasPrice(gasprice)
			// cpu线程数
			threads := ctx.GlobalInt(utils.MinerLegacyThreadsFlag.Name)
			if ctx.GlobalIsSet(utils.MinerThreadsFlag.Name) {
				threads = ctx.GlobalInt(utils.MinerThreadsFlag.Name)
			}
			// 开启挖矿
			if err := ethereum.StartMining(threads); err != nil {
				utils.Fatalf("Failed to start mining: %v", err)
			}
		}
	}