# transaction_signing #

----------
签名者的结构

	type Signer interface {
		// Sender returns the sender address of the transaction.
		// 返回事务发送者地址的方法
		Sender(tx *Transaction) (common.Address, error)
		// SignatureValues returns the raw R, S, V values corresponding to the
		// given signature.
		// 对于给定的签名，返回R,S,V的值
		SignatureValues(tx *Transaction, sig []byte) (r, s, v *big.Int, err error)
		// Hash returns the hash to be signed.
		// 返回对事物签名后的哈希
		Hash(tx *Transaction) common.Hash
		// Equal returns true if the given signer is the same as the receiver.
		// 如果给定的签名者与接收者相同则返回真
		Equal(Signer) bool
	}


签名者分类


 根据给定的链配置和块编号返回签名者。 这里的根据分叉分为三种签名者

	func MakeSigner(config *params.ChainConfig, blockNumber *big.Int) Signer {
		var signer Signer
		switch {
		case config.IsEIP155(blockNumber): //判断是否分叉2675000
			signer = NewEIP155Signer(config.ChainID)
		case config.IsHomestead(blockNumber): //是否是Homestead阶段 1150000
			signer = HomesteadSigner{}
		default:
			signer = FrontierSigner{}
		}
		return signer
	}


SignTx 对事物签名

	func SignTx(tx *Transaction, s Signer, prv *ecdsa.PrivateKey) (*Transaction, error) {
		// 先把事务哈希，随后用私钥签名
		h := s.Hash(tx)
		sig, err := crypto.Sign(h[:], prv)
		if err != nil {
			return nil, err
		}
		// 返回具有签名的事务
		return tx.WithSignature(s, sig)
	}

WithSignature返回具有给定签名的新事务。

	func (tx *Transaction) WithSignature(signer Signer, sig []byte) (*Transaction, error) {
		r, s, v, err := signer.SignatureValues(tx, sig)
		if err != nil {
			return nil, err
		}
		cpy := &Transaction{data: tx.data}
		cpy.data.R, cpy.data.S, cpy.data.V = r, s, v
		return cpy, nil
	}