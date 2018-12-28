# sync #

----------


syncer负责定期与网络同步，下载哈希和块以及处理公告处理程序。

	func (pm *ProtocolManager) syncer() {
		// Start and ensure cleanup of sync mechanisms
		// 启动并确保同步机制的清理
		pm.fetcher.Start()
		defer pm.fetcher.Stop()
		defer pm.downloader.Terminate()
	
		// Wait for different events to fire synchronisation operations
		// 等待不同的事件来触发同步操作
		forceSync := time.NewTicker(forceSyncCycle)
		defer forceSync.Stop()
	
		for {
			select {
			case <-pm.newPeerCh:
				// Make sure we have peers to select from, then sync
				// 确保我们有同行可供选择，然后同步
				// 如果少于5个peer就会不执行同步
				if pm.peers.Len() < minDesiredPeerCount {
					break
				}
				go pm.synchronise(pm.peers.BestPeer())
	
			case <-forceSync.C: //10秒触发一次
				// Force a sync even if not enough peers are present
				// 即使没有足够的对等体，也强制执行同步
				go pm.synchronise(pm.peers.BestPeer())
	
			case <-pm.noMorePeers:
				return
			}
		}
	}