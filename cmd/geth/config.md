# config #

----------
makeConfigNode 加载并应用配置


	func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
		// Load defaults.
		cfg := gethConfig{
			Eth:       eth.DefaultConfig,//包含在以太坊主网上使用的默认设置
			Shh:       whisper.DefaultConfig,//whisper默认设置
			Node:      defaultNodeConfig(),//node的默认设置
			Dashboard: dashboard.DefaultConfig, //dashboard是集成到geth中的数据可视化器，用于收集和可视化以太坊节点的有用信息。
		}
	
		// Load config file.
		// 加载配置文件
		if file := ctx.GlobalString(configFileFlag.Name); file != "" {
			if err := loadConfig(file, &cfg); err != nil {
				utils.Fatalf("%v", err)
			}
		}
	
		// Apply flags.
		// 应用标志
	
		// 应用Node配置
		utils.SetNodeConfig(ctx, &cfg.Node)
		// 建立新的Node
		stack, err := node.New(&cfg.Node)
		if err != nil {
			utils.Fatalf("Failed to create the protocol stack: %v", err)
		}
		// 应用eth配置
		utils.SetEthConfig(ctx, stack, &cfg.Eth)
		if ctx.GlobalIsSet(utils.EthStatsURLFlag.Name) {
			cfg.Ethstats.URL = ctx.GlobalString(utils.EthStatsURLFlag.Name)
		}
		// 应用whisper配置
		utils.SetShhConfig(ctx, stack, &cfg.Shh)
		// 应用Dashboard配置
		utils.SetDashboardConfig(ctx, &cfg.Dashboard)
	
		return stack, cfg
	}

makeFullNode 注册相关服务并返回节点

	func makeFullNode(ctx *cli.Context) *node.Node {
		//加载配置节点
		stack, cfg := makeConfigNode(ctx)
	
		// 注册eth服务（轻服务还是标准）
		utils.RegisterEthService(stack, &cfg.Eth)
	
		// 注册Dashboard服务
		if ctx.GlobalBool(utils.DashboardEnabledFlag.Name) {
			utils.RegisterDashboardService(stack, &cfg.Dashboard, gitCommit)
		}
		// Whisper must be explicitly enabled by specifying at least 1 whisper flag or in dev mode
		// Whisper必须通过指定至少1个Whisper标记或在开发模式下明确启用
		shhEnabled := enableWhisper(ctx)
		shhAutoEnabled := !ctx.GlobalIsSet(utils.WhisperEnabledFlag.Name) && ctx.GlobalIsSet(utils.DeveloperFlag.Name)
		if shhEnabled || shhAutoEnabled {
			if ctx.GlobalIsSet(utils.WhisperMaxMessageSizeFlag.Name) {
				cfg.Shh.MaxMessageSize = uint32(ctx.Int(utils.WhisperMaxMessageSizeFlag.Name))
			}
			if ctx.GlobalIsSet(utils.WhisperMinPOWFlag.Name) {
				cfg.Shh.MinimumAcceptedPOW = ctx.Float64(utils.WhisperMinPOWFlag.Name)
			}
			if ctx.GlobalIsSet(utils.WhisperRestrictConnectionBetweenLightClientsFlag.Name) {
				cfg.Shh.RestrictConnectionBetweenLightClients = true
			}
			utils.RegisterShhService(stack, &cfg.Shh)
		}
	
		// Add the Ethereum Stats daemon if requested.
		// 如果需要，添加以太坊统计守护程序。
		if cfg.Ethstats.URL != "" {
			utils.RegisterEthStatsService(stack, cfg.Ethstats.URL)
		}
		return stack
	}