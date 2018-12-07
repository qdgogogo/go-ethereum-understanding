# accounts #

----------

Wallet电子钱包代表可能包含一个或多个帐户（来自同一种子）的软件或硬件钱包。

	type Wallet interface {
		// URL retrieves the canonical path under which this wallet is reachable. It is
		// user by upper layers to define a sorting order over all wallets from multiple
		// backends.
		// URL检索可以访问此钱包的规范路径。 上层用户可以定义来自多个后端的所有钱包的排序顺序。
		URL() URL
	
		// Status returns a textual status to aid the user in the current state of the
		// wallet. It also returns an error indicating any failure the wallet might have
		// encountered.
		// Status返回文本状态以帮助用户处于钱包的当前状态。 它还会返回一个错误，指示钱包可能遇到的任何故障。
		Status() (string, error)
	
		// Open initializes access to a wallet instance. It is not meant to unlock or
		// decrypt account keys, rather simply to establish a connection to hardware
		// wallets and/or to access derivation seeds.
		//	Open初始化对钱包实例的访问。 它并不意味着解锁或解密帐户密钥，而只是简单地建立与硬件钱包的连接和/或访问派生种子。
		// The passphrase parameter may or may not be used by the implementation of a
		// particular wallet instance. The reason there is no passwordless open method
		// is to strive towards a uniform wallet handling, oblivious to the different
		// backend providers.
		// passphrase参数可以或可以不由特定钱包实例的实现使用。 没有无参数的打开方法的原因是努力实现统一的钱包处理，无视不同的后端提供商。
		//
		// Please note, if you open a wallet, you must close it to release any allocated
		// resources (especially important when working with hardware wallets).
		// 请注意，如果您打开钱包，则必须将其关闭以释放所有已分配的资源（在使用硬件钱包时尤其重要）。
		Open(passphrase string) error
	
		// Close releases any resources held by an open wallet instance.
		// 释放已经打开的钱包实例
		Close() error
	
		// Accounts retrieves the list of signing accounts the wallet is currently aware
		// of. For hierarchical deterministic wallets, the list will not be exhaustive,
		// rather only contain the accounts explicitly pinned during account derivation.
		// Accounts检索钱包当前知道的签名帐户列表。 对于分层确定性钱包，列表不是详尽无遗的，而是仅包含在帐户派生期间明确固定的帐户。
		Accounts() []Account
	
		// Contains returns whether an account is part of this particular wallet or not.
		// Contains返回帐户是否属于此特定钱包的部分。
		Contains(account Account) bool
	
		// Derive attempts to explicitly derive a hierarchical deterministic account at
		// the specified derivation path. If requested, the derived account will be added
		// to the wallet's tracked account list.
		// Derive尝试在指定的派生路径上显式派生层次确定性帐户。 如果需要，派生的帐户将被添加到钱包的跟踪帐户列表中。
		Derive(path DerivationPath, pin bool) (Account, error)
	
		// SelfDerive sets a base account derivation path from which the wallet attempts
		// to discover non zero accounts and automatically add them to list of tracked
		// accounts.
		// SelfDerive设置基本帐户派生路径，钱包从中尝试发现非零帐户，并自动将其添加到已跟踪帐户列表中。
		// Note, self derivaton will increment the last component of the specified path
		// opposed to decending into a child path to allow discovering accounts starting
		// from non zero components.
		// 注意，自派生将递增指定路径的最后一个组件，而不是下降到子路径，以允许从非零组件开始发现帐户。
		// You can disable automatic account discovery by calling SelfDerive with a nil
		// chain state reader.
		// 您可以通过使用nil链状态读取器调用SelfDerive来禁用自动帐户发现。
		SelfDerive(base DerivationPath, chain ethereum.ChainStateReader)
	
		// SignHash requests the wallet to sign the given hash.
		// SignHash请求钱包签署给定的哈希。
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		// 它仅通过其中包含的地址查找指定的帐户，或者可选地借助嵌入的URL字段中的任何位置元数据。
		// If the wallet requires additional authentication to sign the request (e.g.
		// a password to decrypt the account, or a PIN code o verify the transaction),
		// an AuthNeededError instance will be returned, containing infos for the user
		// about which fields or actions are needed. The user may retry by providing
		// the needed details via SignHashWithPassphrase, or by other means (e.g. unlock
		// the account in a keystore).
		// 如果钱包需要额外的身份验证来签署请求（例如，用于解密帐户的密码，或PIN码以验证交易），则将返回AuthNeededError实例，其包含用户关于需要哪些字段或动作的信息。
		// 用户可以通过SignHashWithPassphrase或其他方式（例如，在密钥库中解锁帐户）提供所需的详细信息来重试。
		SignHash(account Account, hash []byte) ([]byte, error)
	
		// SignTx requests the wallet to sign the given transaction.
		//
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		//
		// If the wallet requires additional authentication to sign the request (e.g.
		// a password to decrypt the account, or a PIN code to verify the transaction),
		// an AuthNeededError instance will be returned, containing infos for the user
		// about which fields or actions are needed. The user may retry by providing
		// the needed details via SignTxWithPassphrase, or by other means (e.g. unlock
		// the account in a keystore).
		//SignTx请求钱包签署给定的交易。
		//它仅通过其中包含的地址查找指定的帐户，或者可选地借助嵌入的URL字段中的任何位置元数据。
		//如果钱包需要额外的身份验证来签署请求（例如，用于解密帐户的密码或用于验证交易的PIN码），则将返回AuthNeededError实例，其包含用户关于需要哪些字段或动作的信息。
		//用户可以通过SignTxWithPassphrase或其他方式（例如，在密钥库中解锁帐户）提供所需的详细信息来重试。
		SignTx(account Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
	
		// SignHashWithPassphrase requests the wallet to sign the given hash with the
		// given passphrase as extra authentication information.
		// SignHashWithPassphrase请求钱包使用给定的密码作为额外的身份验证信息对给定的哈希进行签名。
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		// 它仅通过其中包含的地址查找指定的帐户，或者可选地借助嵌入的URL字段中的任何位置元数据。
		SignHashWithPassphrase(account Account, passphrase string, hash []byte) ([]byte, error)
	
		// SignTxWithPassphrase requests the wallet to sign the given transaction, with the
		// given passphrase as extra authentication information.
		// SignTxWithPassphrase请求钱包签署给定的事务，给定的密码短语作为额外的身份验证信息。
		// It looks up the account specified either solely via its address contained within,
		// or optionally with the aid of any location metadata from the embedded URL field.
		// 它仅通过其中包含的地址查找指定的帐户，或者可选地借助嵌入的URL字段中的任何位置元数据。
		SignTxWithPassphrase(account Account, passphrase string, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
	}





Backend是一个“钱包提供者”，可能包含一批他们可以根据请求签署交易的账户，



	type Backend interface {
		// Wallets retrieves the list of wallets the backend is currently aware of.
		//
		// The returned wallets are not opened by default. For software HD wallets this
		// means that no base seeds are decrypted, and for hardware wallets that no actual
		// connection is established.
		//
		// The resulting wallet list will be sorted alphabetically based on its internal
		// URL assigned by the backend. Since wallets (especially hardware) may come and
		// go, the same wallet might appear at a different positions in the list during
		// subsequent retrievals.
		// 钱包检索后端当前知道的钱包列表。 默认情况下不会打开返回的钱包。 对于软件HD钱包，这意味着没有基础种子被解密，并且对于硬件钱包并没有建立实际连接。
		// 生成的钱包列表将根据后端分配的内部URL按字母顺序排序。 由于钱包（尤其是硬件）可能来来往往，因此在后续检索期间，相同的钱包可能会出现在列表中的不同位置。
		Wallets() []Wallet
	
		// Subscribe creates an async subscription to receive notifications when the
		// backend detects the arrival or departure of a wallet.
		// 订阅创建异步订阅，以在后端检测到钱包的到达或离开时接收通知。
		Subscribe(sink chan<- WalletEvent) event.Subscription
	}