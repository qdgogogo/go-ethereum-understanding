# block #

----------

块Block是区块链基本的存储结构

由块头和块体构成

首先整个区块的结构如下，


	type Block struct {
		header       *Header  //块头
		uncles       []*Header //叔块块头
		transactions Transactions //事务集合
	
		// caches
		hash atomic.Value	// 块哈希
		size atomic.Value	// 块大小
	
		// Td is used by package core to store the total difficulty
		// of the chain up to and including the block.
		td *big.Int	//难度
	
		// These fields are used by package eth to track
		// inter-peer block relay.
		ReceivedAt   time.Time	//时间戳
		ReceivedFrom interface{} //从那个节点收到的
	}

块头的结构

	type Header struct {
		ParentHash  common.Hash    `json:"parentHash"       gencodec:"required"` //指向前一个块
		UncleHash   common.Hash    `json:"sha3Uncles"       gencodec:"required"` //叔块集合
		Coinbase    common.Address `json:"miner"            gencodec:"required"` //币基账户，就是谁挖出的这个块
		Root        common.Hash    `json:"stateRoot"        gencodec:"required"` //状态根
		TxHash      common.Hash    `json:"transactionsRoot" gencodec:"required"` //事务根
		ReceiptHash common.Hash    `json:"receiptsRoot"     gencodec:"required"` //收据根
		Bloom       Bloom          `json:"logsBloom"        gencodec:"required"` //布隆过滤器
		Difficulty  *big.Int       `json:"difficulty"       gencodec:"required"` //难度
		Number      *big.Int       `json:"number"           gencodec:"required"` //块高度
		GasLimit    uint64         `json:"gasLimit"         gencodec:"required"` //块中至少需要消耗的gas数
		GasUsed     uint64         `json:"gasUsed"          gencodec:"required"` //gas使用
		Time        *big.Int       `json:"timestamp"        gencodec:"required"` //时间戳
		Extra       []byte         `json:"extraData"        gencodec:"required"` //附带数据
		MixDigest   common.Hash    `json:"mixHash"          gencodec:"required"` //用于验证的数据
		Nonce       BlockNonce     `json:"nonce"            gencodec:"required"` //挖矿的随机数，计算量证明
	}

header 的Hash()方法 返回header的rlpHash
	
	func (h *Header) Hash() common.Hash {
		return rlpHash(h)
	}

块体结构

块体body是简单的数据容器，将内容整合在块中，块并没有明确的将块头块体隔离，块体可以通过方法获得

	func (b *Block) Body() *Body { return &Body{b.transactions, b.uncles} }

	
	type Body struct {
		Transactions []*Transaction //事务集合
		Uncles       []*Header		//叔块集合
	}


