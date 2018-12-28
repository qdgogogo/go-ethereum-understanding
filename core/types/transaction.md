# transaction #

----------

事务类型的定义


	type Transaction struct {
		data txdata  //事务具体数据
		// caches
		hash atomic.Value  	// 事务的哈希
		size atomic.Value	//事务的大小
		from atomic.Value	//来源
	}
	
	type txdata struct {
		AccountNonce uint64          `json:"nonce"    gencodec:"required"` //此交易的发送者已发送过的交易数
		Price        *big.Int        `json:"gasPrice" gencodec:"required"` //此交易的gas price
		GasLimit     uint64          `json:"gas"      gencodec:"required"`	//本交易允许消耗的最大gas数量
		Recipient    *common.Address `json:"to"       rlp:"nil"` // 交易的接收者，是一个地址，为空代表合约创建
		Amount       *big.Int        `json:"value"    gencodec:"required"`//交易转移的以太币数量，单位是wei
		Payload      []byte          `json:"input"    gencodec:"required"`//交易可以携带的数据，在不同类型的交易中有不同的含义
	
		// Signature values
		// 签名
		V *big.Int `json:"v" gencodec:"required"`
		R *big.Int `json:"r" gencodec:"required"`
		S *big.Int `json:"s" gencodec:"required"`
	
		// This is only used when marshaling to JSON.
		Hash *common.Hash `json:"hash" rlp:"-"`
	}


创建新事务

	func NewTransaction(nonce uint64, to common.Address, amount *big.Int, gasLimit uint64, gasPrice *big.Int, data []byte) *Transaction {
		return newTransaction(nonce, &to, amount, gasLimit, gasPrice, data)
	}

	func newTransaction(nonce uint64, to *common.Address, amount *big.Int, gasLimit uint64, gasPrice *big.Int, data []byte) *Transaction {
		if len(data) > 0 {
			data = common.CopyBytes(data)
		}
		d := txdata{
			AccountNonce: nonce,
			Recipient:    to,
			Payload:      data,
			Amount:       new(big.Int),
			GasLimit:     gasLimit,
			Price:        new(big.Int),
			V:            new(big.Int),
			R:            new(big.Int),
			S:            new(big.Int),
		}
		if amount != nil {
			d.Amount.Set(amount)
		}
		if gasPrice != nil {
			d.Price.Set(gasPrice)
		}
		// 返回一个未签名的事务
		return &Transaction{data: d}
	}

黄皮书上解释的，事务transaction和消息message区别：

transaction：由外部人员签名的一段数据，它表示消息或新的自治对象。 交易记录在区块链的每个区块中。也就是所事务可以代表一个消息或者合约。

message：通过自治对象的确定性操作或事务的加密安全签名在两个帐户之间传递的数据（作为一组字节）和值（指定为以太）。这意味着消息是在两个帐户之间传递的以太坊的数据和值。 通过彼此交互的合同或通过交易创建消息。 

总的来说事务是明确在区块链上的而消息是“内部的”。


AsMessage将事务作为消息输出，需要一个签名者


	func (tx *Transaction) AsMessage(s Signer) (Message, error) {
		msg := Message{
			nonce:      tx.data.AccountNonce,
			gasLimit:   tx.data.GasLimit,
			gasPrice:   new(big.Int).Set(tx.data.Price),
			to:         tx.data.Recipient,
			amount:     tx.data.Amount,
			data:       tx.data.Payload,
			checkNonce: true,
		}
	
		var err error
		msg.from, err = Sender(s, tx)
		return msg, err
	}