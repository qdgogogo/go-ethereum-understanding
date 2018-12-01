# nat #

----------

Nat 定义，类型，别人[博客](https://blog.csdn.net/freeking101/article/details/77962312)总结的不错。

另外upnp和pmp协议的作用，以及udp协议再p2p网络中的打洞方式，也有人写出了不错的总结。

转载自[github](https://github.com/ZtesoftCS/go-ethereum-code-analysis/blob/master/p2p-nat%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.md#p2p%E4%B8%AD%E7%9A%84udp%E5%8D%8F%E8%AE%AE)

Map在m上添加端口映射并保持活动直到c关闭。
	
	func Map(m Interface, c chan struct{}, protocol string, extport, intport int, name string) {
		log := log.New("proto", protocol, "extport", extport, "intport", intport, "interface", m)
		refresh := time.NewTimer(mapUpdateInterval)
		defer func() {
			refresh.Stop()
			log.Debug("Deleting port mapping")
			m.DeleteMapping(protocol, extport, intport)
		}()
		if err := m.AddMapping(protocol, extport, intport, name, mapTimeout); err != nil {
			log.Debug("Couldn't add port mapping", "err", err)
		} else {
			log.Info("Mapped network port")
		}
		for {
			select {
			case _, ok := <-c:
				if !ok {
					return
				}
			case <-refresh.C:
				log.Trace("Refreshing port mapping")
				if err := m.AddMapping(protocol, extport, intport, name, mapTimeout); err != nil {
					log.Debug("Couldn't add port mapping", "err", err)
				}
				refresh.Reset(mapUpdateInterval)
			}
		}
	}
