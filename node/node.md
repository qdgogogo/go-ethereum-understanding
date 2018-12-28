# node #

----------
**结构体**

Node 配置Config

Config表示一小组配置值，用于微调协议栈的P2P网络层。 所有注册的服务都可以进一步扩展这些值。


	type Config struct {
		// Name设置节点的实例名称。 它不能包含/字符，并在devp2p节点标识符中使用。 geth的实例名称是“geth”。 如果未指定任何值，则使用当前可执行文件的基名。
		Name string `toml:"-"`
	
		// UserIdent（如果已设置）用作devp2p节点标识符中的附加组件。
		UserIdent string `toml:",omitempty"`
	
		// 应设置为程序的版本号。 它用在devp2p节点标识符中。
		Version string `toml:"-"`
	
		// DataDir是节点应该用于任何数据存储要求的文件系统文件夹。 配置的数据目录不会直接与已注册的服务共享，而是可以使用实用程序方法来创建/访问数据库或平面文件。 这使得能够完全驻留在存储器中的短暂节点成为可能。
		DataDir string
	
		// p2p的配置
		P2P p2p.Config
	
		// KeyStoreDir是包含私钥的文件系统文件夹。 可以将目录指定为相对路径，在这种情况下，它将相对于当前目录进行解析。
		//
		// 如果KeyStoreDir为空，则默认位置是DataDir的“keystore”子目录。 如果DataDir未指定且KeyStoreDir为空，则由New创建临时目录，并在节点停止时销毁。
		KeyStoreDir string `toml:",omitempty"`
	
		// 以牺牲安全性为代价降低了密钥库scrypt KDF的内存和CPU要求。
		UseLightweightKDF bool `toml:",omitempty"`
	
		// 禁用硬件钱包监控和连接。
		NoUSB bool `toml:",omitempty"`
	
		// IPCPath是放置IPC端点的请求位置。 如果路径是简单文件名，则将其放在数据目录内（或Windows上的根管道路径上），而如果它是可解析的路径名（绝对路径名或相对路径名），则强制执行该特定路径。 空路径禁用IPC。
		IPCPath string `toml:",omitempty"`
	
		// HTTPHost是启动HTTP RPC服务器的主机接口。 如果此字段为空，则不会启动任何HTTP API端点。
		HTTPHost string `toml:",omitempty"`
	
		// HTTPPort是启动HTTP RPC服务器的TCP端口号。 默认的零值是/ valid，将随机选择一个端口号（对于临时节点很有用）。
		HTTPPort int `toml:",omitempty"`
	
		// HTTPCors是发送给请求客户端的跨源资源共享标头。 请注意，CORS是一种浏览器强制安全性，它对于自定义HTTP客户端完全没用。
		HTTPCors []string `toml:",omitempty"`
	
		// HTTPVirtualHosts是传入请求允许的虚拟主机名列表。 这是默认情况下{'localhost'}。 使用它可以防止像DNS重新绑定这样的攻击，它通过简单地伪装成在同一个源内来绕过SOP。 这些攻击不使用CORS，因为它们不是跨域的。 通过显式检查Host-header，服务器将不允许针对具有恶意主机域的服务器发出的请求。 直接使用IP地址的请求不受影响
		HTTPVirtualHosts []string `toml:",omitempty"`
	
		// HTTPModules是要通过HTTP RPC接口公开的API模块列表。 如果模块列表为空，则将公开指定为public的所有RPC API端点。
		HTTPModules []string `toml:",omitempty"`
	
		// HTTPTimeouts允许自定义HTTP RPC接口使用的超时值。
		HTTPTimeouts rpc.HTTPTimeouts
	
		// WSHost是启动websocket RPC服务器的主机接口。 如果此字段为空，则不会启动websocket API端点。
		WSHost string `toml:",omitempty"`
	
		// WSPort是启动websocket RPC服务器的TCP端口号。 默认的零值是/ valid，将随机选择一个端口号（对于临时节点很有用）。
		WSPort int `toml:",omitempty"`
	
		// WSOrigins是接受来自websocket请求的域列表。 请注意，服务器只能根据客户端发送的HTTP请求执行操作，并且无法验证请求标头的有效性。
		WSOrigins []string `toml:",omitempty"`
	
		// WSModules是通过websocket RPC接口公开的API模块列表。 如果模块列表为空，则将公开指定为public的所有RPC API端点。
		WSModules []string `toml:",omitempty"`
	
		// WSExposeAll通过WebSocket RPC接口而不仅仅是公共接口公开所有API模块。
		//
		// *警告*仅在节点在受信任网络中运行时才设置此项，向不受信任的用户公开私有API是一个主要的安全风险。
		WSExposeAll bool `toml:",omitempty"`
	
		// Logger是一个与p2p.Server一起使用的自定义日志。
		Logger log.Logger `toml:",omitempty"`
	}


Node

New创建了一个新的P2P节点，为协议注册做好了准备。
	
	func New(conf *Config) (*Node, error) {
		// 复制config并解析datadir，以便将来对当前工作目录的更改不会影响节点。
		confCopy := *conf
		conf = &confCopy
		if conf.DataDir != "" {
			absdatadir, err := filepath.Abs(conf.DataDir)
			if err != nil {
				return nil, err
			}
			conf.DataDir = absdatadir
		}
		// 确保实例名称不会与数据目录中的其他文件发生奇怪的冲突。
		if strings.ContainsAny(conf.Name, `/\`) {
			return nil, errors.New(`Config.Name must not contain '/' or '\'`)
		}
		if conf.Name == datadirDefaultKeyStore {
			return nil, errors.New(`Config.Name cannot be "` + datadirDefaultKeyStore + `"`)
		}
		if strings.HasSuffix(conf.Name, ".ipc") {
			return nil, errors.New(`Config.Name cannot end in ".ipc"`)
		}
		// 确保AccountManager方法在节点启动之前工作。 我们在cmd / geth中依赖于此。
		am, ephemeralKeystore, err := makeAccountManager(conf)
		if err != nil {
			return nil, err
		}
		if conf.Logger == nil {
			conf.Logger = log.New()
		}
		//注意：与Config的任何交互都会在数据目录或实例目录中创建/触摸文件，直到Start为止。
		//touch files是什么意思？
		return &Node{
			accman:            am,
			ephemeralKeystore: ephemeralKeystore,
			config:            conf,
			serviceFuncs:      []ServiceConstructor{},
			ipcEndpoint:       conf.IPCEndpoint(),
			httpEndpoint:      conf.HTTPEndpoint(),
			wsEndpoint:        conf.WSEndpoint(),
			//eventmux:          new(event.TypeMux),
			log:               conf.Logger,
		}, nil
	}

Attach创建一个附加到进程内API处理程序的RPC客户端。


	func (n *Node) Attach() (*rpc.Client, error) {
		n.lock.RLock()
		defer n.lock.RUnlock()
	
		if n.server == nil {
			return nil, ErrNodeStopped
		}
		return rpc.DialInProc(n.inprocHandler), nil
	}

创建p2p节点并启动它

	func (n *Node) Start() error {
		n.lock.Lock()
		defer n.lock.Unlock()
	
		// Short circuit if the node's already running
		// 如果节点已经在运行则短路
		if n.server != nil {
			return ErrNodeRunning
		}
		if err := n.openDataDir(); err != nil {
			return err
		}
	
		// Initialize the p2p server. This creates the node key and
		// discovery databases.
		// 初始化p2p服务器。 这将创建节点密钥和发现数据库。
		n.serverConfig = n.config.P2P
		n.serverConfig.PrivateKey = n.config.NodeKey()
		n.serverConfig.Name = n.config.NodeName()
		n.serverConfig.Logger = n.log
		if n.serverConfig.StaticNodes == nil {
			n.serverConfig.StaticNodes = n.config.StaticNodes()
		}
		if n.serverConfig.TrustedNodes == nil {
			n.serverConfig.TrustedNodes = n.config.TrustedNodes()
		}
		if n.serverConfig.NodeDatabase == "" {
			n.serverConfig.NodeDatabase = n.config.NodeDB()
		}
		running := &p2p.Server{Config: n.serverConfig}
		n.log.Info("Starting peer-to-peer node", "instance", n.serverConfig.Name)
	
		// Otherwise copy and specialize the P2P configuration
		// 否则复制并专门化P2P配置
		services := make(map[reflect.Type]Service)
		for _, constructor := range n.serviceFuncs {
			// Create a new context for the particular service
			// 为特定服务创建新环境
			ctx := &ServiceContext{
				config:         n.config,
				services:       make(map[reflect.Type]Service),
				EventMux:       n.eventmux,
				AccountManager: n.accman,
			}
			for kind, s := range services { // copy needed for threaded access
				ctx.services[kind] = s
			}
			// Construct and save the service
			// 构建并保存服务
			service, err := constructor(ctx)
			if err != nil {
				return err
			}
			kind := reflect.TypeOf(service)
			if _, exists := services[kind]; exists {
				return &DuplicateServiceError{Kind: kind}
			}
			services[kind] = service
		}
		// Gather the protocols and start the freshly assembled P2P server
		// 收集协议并启动新组装的P2P服务器
	
		for _, service := range services {
			running.Protocols = append(running.Protocols, service.Protocols()...)
		}
		if err := running.Start(); err != nil {
			return convertFileLockError(err)
		}
		// Start each of the services
		// 启动每个服务
		started := []reflect.Type{}
		for kind, service := range services {
			// Start the next service, stopping all previous upon failure
			// 启动下一个服务，在失败时停止所有服务
			if err := service.Start(running); err != nil {
				for _, kind := range started {
					services[kind].Stop()
				}
				running.Stop()
	
				return err
			}
			// Mark the service started for potential cleanup
			// 标记为潜在清理启动的服务
			started = append(started, kind)
		}
		// Lastly start the configured RPC interfaces
		// 最后启动配置的RPC接口
		if err := n.startRPC(services); err != nil {
			for _, service := range services {
				service.Stop()
			}
			running.Stop()
			return err
		}
		// Finish initializing the startup
		n.services = services
		n.server = running
		n.stop = make(chan struct{})
	
		return nil
	}