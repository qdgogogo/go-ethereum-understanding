# api #

----------

发送事务的参数结构

	type SendTxArgs struct {
		From     common.Address  `json:"from"`
		To       *common.Address `json:"to"`
		Gas      *hexutil.Uint64 `json:"gas"`
		GasPrice *hexutil.Big    `json:"gasPrice"`
		Value    *hexutil.Big    `json:"value"`
		Nonce    *hexutil.Uint64 `json:"nonce"`
		// We accept "data" and "input" for backwards-compatibility reasons. "input" is the
		// newer name and should be preferred by clients.
		Data  *hexutil.Bytes `json:"data"`
		Input *hexutil.Bytes `json:"input"`
	}

SendTransaction将根据给定的参数创建一个事务，并尝试使用与args.To关联的密钥对其进行签名。 如果给定的passwd无法解密密钥，则会失败。

	func (s *PrivateAccountAPI) SendTransaction(ctx context.Context, args SendTxArgs, passwd string) (common.Hash, error) {
		if args.Nonce == nil {
			// Hold the addresse's mutex around signing to prevent concurrent assignment of
			// the same nonce to multiple accounts.
			// 保持addresse的互斥锁围绕签名以防止并发分配
			s.nonceLock.LockAddr(args.From)
			defer s.nonceLock.UnlockAddr(args.From)
		}
		// 返回已签名的事务
		signed, err := s.signTransaction(ctx, &args, passwd)
		if err != nil {
			log.Warn("Failed transaction send attempt", "from", args.From, "to", args.To, "value", args.Value.ToInt(), "err", err)
			return common.Hash{}, err
		}
		// 提交到事务池
		return submitTransaction(ctx, s.b, signed)
	}

组装事务

	func (args *SendTxArgs) toTransaction() *types.Transaction {
		var input []byte
		if args.Data != nil {
			input = *args.Data
		} else if args.Input != nil {
			input = *args.Input
		}
		if args.To == nil {
			return types.NewContractCreation(uint64(*args.Nonce), (*big.Int)(args.Value), uint64(*args.Gas), (*big.Int)(args.GasPrice), input)
		}
		return types.NewTransaction(uint64(*args.Nonce), *args.To, (*big.Int)(args.Value), uint64(*args.Gas), (*big.Int)(args.GasPrice), input)
	}