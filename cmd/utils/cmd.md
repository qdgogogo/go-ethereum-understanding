# cmd #

----------
启动节点和关闭节点
	
	func StartNode(stack *node.Node) {
		if err := stack.Start(); err != nil {
			Fatalf("Error starting protocol stack: %v", err)
		}
		go func() {
			sigc := make(chan os.Signal, 1)
			// Notify函数让signal包将输入信号转发到c。
			signal.Notify(sigc, syscall.SIGINT, syscall.SIGTERM)
			defer signal.Stop(sigc)
			//阻塞获取信号
			<-sigc
			log.Info("Got interrupt, shutting down...")
			// 停止节点
			go stack.Stop()
			for i := 10; i > 0; i-- {
				<-sigc
				if i > 1 {
					log.Warn("Already shutting down, interrupt more to panic.", "times", i-1)
				}
			}
			debug.Exit() // ensure trace and CPU profile data is flushed.
			debug.LoudPanic("boom")
		}()
	}