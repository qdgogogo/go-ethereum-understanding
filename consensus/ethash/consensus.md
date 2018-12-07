# consensus #

----------


实现共识引擎，累积块和叔叔奖励，设置最终状态并组装块。

	func (ethash *Ethash) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header, receipts []*types.Receipt) (*types.Block, error) {
		// Accumulate any block and uncle rewards and commit the final state root
		// 累积任何块和叔块奖励并提交最终状态根
		accumulateRewards(chain.Config(), state, header, uncles)
		header.Root = state.IntermediateRoot(chain.Config().IsEIP158(header.Number))
	
		// Header seems complete, assemble into a block and return
		// 标题似乎完整，组装成一个块并返回
		return types.NewBlock(header, txs, uncles, receipts), nil
	}




将给定块的coinbase与挖掘奖励相关联。 总奖励包括静态区块奖励和包含的叔叔的奖励。 每个叔叔街区的coinbase也会得到奖励。

	func accumulateRewards(config *params.ChainConfig, state *state.StateDB, header *types.Header, uncles []*types.Header) {
		// Select the correct block reward based on chain progression
		// 根据区块链的分叉选择正确的区块奖励
		blockReward := FrontierBlockReward
		if config.IsByzantium(header.Number) {
			blockReward = ByzantiumBlockReward
		}
		if config.IsConstantinople(header.Number) {
			blockReward = ConstantinopleBlockReward
		}
		// Accumulate the rewards for the miner and any included uncles
		// 计算叔块获得的奖励
		reward := new(big.Int).Set(blockReward)
		r := new(big.Int)
		for _, uncle := range uncles {
			r.Add(uncle.Number, big8)
			r.Sub(r, header.Number)
			r.Mul(r, blockReward)
			r.Div(r, big8)
			state.AddBalance(uncle.Coinbase, r)
	
			r.Div(blockReward, big32)
			reward.Add(reward, r)
		}
		// 计算块的奖励
		state.AddBalance(header.Coinbase, reward)
	}