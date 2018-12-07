# manager #

----------
Manager是一个总体账户经理，可以与各种后端进行通信以进行签名交易。
	
	type Manager struct {
		// 目前已注册的后端索引
		backends map[reflect.Type][]Backend // Index of backends currently registered
		// 电子钱包更新所有后端的订阅
		updaters []event.Subscription       // Wallet update subscriptions for all backends
		// 用于后端钱包的订阅接收器更改
		updates  chan WalletEvent           // Subscription sink for backend wallet changes
		// 所有已注册的后端钱包的缓存
		wallets  []Wallet                   // Cache of all wallets from all registered backends
	
		// 电子钱包通知到达/离开
		feed event.Feed // Wallet feed notifying of arrivals/departures
	
		quit chan chan error
		lock sync.RWMutex
	}

NewManager创建一个通用帐户管理器，通过各种支持的后端签署事务。

	
	func NewManager(backends ...Backend) *Manager {
		// Retrieve the initial list of wallets from the backends and sort by URL
		// 从后端检索钱包的初始列表并按URL排序
		var wallets []Wallet
		for _, backend := range backends {
			wallets = merge(wallets, backend.Wallets()...)
		}
		// Subscribe to wallet notifications from all backends
		// 订阅来自所有后端的钱包通知
		updates := make(chan WalletEvent, 4*len(backends))
	
		subs := make([]event.Subscription, len(backends))
		// 便利给每个后端支持配置通知管道
		for i, backend := range backends {
			subs[i] = backend.Subscribe(updates)
		}
		// Assemble the account manager and return
		// 组装账户经理并返回
		am := &Manager{
			backends: make(map[reflect.Type][]Backend),
			updaters: subs,
			updates:  updates,
			wallets:  wallets,
			quit:     make(chan chan error),
		}
		// 配置后端类型映射
		for _, backend := range backends {
			kind := reflect.TypeOf(backend)
			am.backends[kind] = append(am.backends[kind], backend)
		}
		//钱包事件循环，用于监听来自后端的通知并更新钱包缓存。
		go am.update()
	
		return am
	}


update是钱包事件循环，用于侦听来自后端的通知并更新钱包缓存。

	func (am *Manager) update() {
		// Close all subscriptions when the manager terminates
		// 管理器终止时关闭所有订阅
		defer func() {
			am.lock.Lock()
			for _, sub := range am.updaters {
				sub.Unsubscribe()
			}
			am.updaters = nil
			am.lock.Unlock()
		}()
	
		// Loop until termination
		// 循环监听事件
		for {
			select {
			case event := <-am.updates:
				// Wallet event arrived, update local cache
				// 钱包事件到了，更新了本地缓存
				am.lock.Lock()
				// 判断钱包到达或离开，并设置wallets
				switch event.Kind {
				case WalletArrived: //检测到硬件钱包或文件系统中的keystore
					am.wallets = merge(am.wallets, event.Wallet)
				case WalletDropped:
					am.wallets = drop(am.wallets, event.Wallet)
				}
				am.lock.Unlock()
	
				// Notify any listeners of the event
				// 向监听者通知该事件
				am.feed.Send(event)
	
			case errc := <-am.quit:
				// Manager terminating, return
				errc <- nil
				return
			}
		}
	}