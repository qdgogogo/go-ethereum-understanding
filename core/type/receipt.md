# receipt #

----------
Receipt代表事务的结果，事务的输出

	type Receipt struct {
		// Consensus fields
		PostState         []byte `json:"root"`  //保存了创建该Receipt对象时，整个Block内所有“帐户”的当时状态。
		Status            uint64 `json:"status"`
		CumulativeGasUsed uint64 `json:"cumulativeGasUsed" gencodec:"required"` //在块中执行此事务时使用的总gas量。
		// Ethereum内部实现的一个256bit长Bloom Filter,这里Receipt的Bloom，被用以验证某个给定的Log是否处于Receipt已有的Log数组中。
		Bloom             Bloom  `json:"logsBloom"         gencodec:"required"`
		Logs              []*Log `json:"logs"              gencodec:"required"` //此事务生成的日志对象数组。
	
		// Implementation fields (don't reorder!)
		TxHash          common.Hash    `json:"transactionHash" gencodec:"required"`
		ContractAddress common.Address `json:"contractAddress"`
		GasUsed         uint64         `json:"gasUsed" gencodec:"required"` //仅此特定交易所使用的gas量。
	}
