# flags #

----------

RegisterEthService将以太坊客户端添加到堆栈。

	func RegisterEthService(stack *node.Node, cfg *eth.Config) {
		var err error
		if cfg.SyncMode == downloader.LightSync { //下载是轻节点模式
			// 注册LightEthereum服务
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				return les.New(ctx, cfg)
			})
		} else {
			// 注册标准Ethereum服务
			// 在这里注册的构造方法将会在node.start()中添加服务并保存
			err = stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
				// 创建eth对象
				fullNode, err := eth.New(ctx, cfg)
				if fullNode != nil && cfg.LightServ > 0 {
					// 默认LightServ的大小是0 也就是不会启动LesServer
					// LesServer是给轻量级节点提供服务的。
					ls, _ := les.NewLesServer(fullNode, cfg)
					fullNode.AddLesServer(ls)
				}
				return fullNode, err
			})
		}
		if err != nil {
			Fatalf("Failed to register the Ethereum service: %v", err)
		}
	}