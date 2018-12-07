# config #

----------
创建账户管理，密钥等
	
	func makeAccountManager(conf *Config) (*accounts.Manager, string, error) {
		scryptN, scryptP, keydir, err := conf.AccountConfig()
		var ephemeral string
		if keydir == "" {
			// There is no datadir.
			keydir, err = ioutil.TempDir("", "go-ethereum-keystore")
			ephemeral = keydir
		}
	
		if err != nil {
			return nil, "", err
		}
		// 如果没有则创建keydir目录
		if err := os.MkdirAll(keydir, 0700); err != nil {
			return nil, "", err
		}
		// Assemble the account manager and supported backends
		// 组装账户管理和支持的后端
		backends := []accounts.Backend{
			keystore.NewKeyStore(keydir, scryptN, scryptP),
		}
		// 硬钱包
		if !conf.NoUSB {
			// Start a USB hub for Ledger hardware wallets
			if ledgerhub, err := usbwallet.NewLedgerHub(); err != nil {
				log.Warn(fmt.Sprintf("Failed to start Ledger hub, disabling: %v", err))
			} else {
				backends = append(backends, ledgerhub)
			}
			// Start a USB hub for Trezor hardware wallets
			if trezorhub, err := usbwallet.NewTrezorHub(); err != nil {
				log.Warn(fmt.Sprintf("Failed to start Trezor hub, disabling: %v", err))
			} else {
				backends = append(backends, trezorhub)
			}
		}
		return accounts.NewManager(backends...), ephemeral, nil
	}


AccountConfig确定scrypt和keydirectory的设置


	func (c *Config) AccountConfig() (int, int, string, error) {
		// 按配置选择标准的加密算法参数设置还是轻量级的算法参数设置，两者的区别在于轻量级的占用cpu时间少
		scryptN := keystore.StandardScryptN
		scryptP := keystore.StandardScryptP
		if c.UseLightweightKDF {
			scryptN = keystore.LightScryptN
			scryptP = keystore.LightScryptP
		}
	
		var (
			keydir string
			err    error
		)
		//设置密钥存放目录
		switch {
		case filepath.IsAbs(c.KeyStoreDir):
			keydir = c.KeyStoreDir
		case c.DataDir != "":
			if c.KeyStoreDir == "" {
				keydir = filepath.Join(c.DataDir, datadirDefaultKeyStore)
			} else {
				keydir, err = filepath.Abs(c.KeyStoreDir)
			}
		case c.KeyStoreDir != "":
			keydir, err = filepath.Abs(c.KeyStoreDir)
		}
		return scryptN, scryptP, keydir, err
	}