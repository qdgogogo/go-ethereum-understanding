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