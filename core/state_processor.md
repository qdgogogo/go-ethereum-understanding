# state_processor #

----------

StateProcessor状态处理器是最基本的处理器，用来将事务状态从一个状态改变到另一个状态

	type StateProcessor struct {
		config *params.ChainConfig // 链的的配置文件
		bc     *BlockChain         // 规范的链
		engine consensus.Engine    // 共识引擎
	}


初始化一个新的状态处理器

	func NewStateProcessor(config *params.ChainConfig, bc *BlockChain, engine consensus.Engine) *StateProcessor {
		return &StateProcessor{
			config: config,
			bc:     bc,
			engine: engine,
		}
	}


Process方法方法根据以太坊的规则改变状态，通过运行事务消息，将奖励应用到coinbase和包含的叔块。

Process方法返回流程中累积的收据和日志，并返回流程中使用的气体量。 如果任何交易因气体不足而未能执行，则会返回错误。

	func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
		var (
			receipts types.Receipts
			usedGas  = new(uint64)
			header   = block.Header() //是以区块为单位处理的
			allLogs  []*types.Log
			gp       = new(GasPool).AddGas(block.GasLimit())
		)
		// Mutate the block and state according to any hard-fork specs
		// 根据硬分叉的规范改变块和状态
		// 根据DAO硬分叉规则修改状态数据库，将一组DAO帐户的所有余额转移到单个退款合同。
		if p.config.DAOForkSupport && p.config.DAOForkBlock != nil && p.config.DAOForkBlock.Cmp(block.Number()) == 0 {
			misc.ApplyDAOHardFork(statedb)
		}
		// Iterate over and process the individual transactions
		// 迭代并处理各个事务
		for i, tx := range block.Transactions() {
			// 配置EVM需要的准备数据
			statedb.Prepare(tx.Hash(), block.Hash(), i)
			// 执行事务
			receipt, _, err := ApplyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, usedGas, cfg)
			if err != nil {
				return nil, nil, 0, err
			}
			receipts = append(receipts, receipt)
			allLogs = append(allLogs, receipt.Logs...)
		}
		// Finalize the block, applying any consensus engine specific extras (e.g. block rewards)
		// 最终确定块，应用任何共识引擎特定的额外内容（例如块奖励）
		p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles(), receipts)
	
		return receipts, allLogs, *usedGas, nil
	}


ApplyTransaction尝试将事务应用于给定的状态数据库，并将输入参数用于其环境。 它返回事务的收据，使用的气体，如果事务失败则返回错误，表示块无效。

也就是说这是对单个事务的运行处理，Process是对块运行的处理

	func ApplyTransaction(config *params.ChainConfig, bc ChainContext, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, uint64, error) {
		// 判断签名者的类型，并将事务作为消息返回
		msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
		if err != nil {
			return nil, 0, err
		}
		// Create a new context to be used in the EVM environment
		// 创建新的虚拟机运行环境
		context := NewEVMContext(msg, header, bc, author)
		// Create a new environment which holds all relevant information
		// about the transaction and calling mechanisms.
		// 创建一个新环境，其中包含有关事务和调用机制的所有相关信息。
		vmenv := vm.NewEVM(context, statedb, config, cfg)
		// Apply the transaction to the current state (included in the env)
		// 应用事务到当前状态
		_, gas, failed, err := ApplyMessage(vmenv, msg, gp)
		if err != nil {
			return nil, 0, err
		}
		// Update the state with pending changes
		// 更新状态
		var root []byte
		if config.IsByzantium(header.Number) {
			statedb.Finalise(true)
		} else {
			root = statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()
		}
		*usedGas += gas
	
		// Create a new receipt for the transaction, storing the intermediate root and gas used by the tx
		// based on the eip phase, we're passing whether the root touch-delete accounts.
		// 为事务创建一个新收据，在eip 阶段存储中间root，和事务使用的gas,我们正在正在传递root touch-delete 账户
		receipt := types.NewReceipt(root, failed, *usedGas)
		receipt.TxHash = tx.Hash()
		receipt.GasUsed = gas
		// if the transaction created a contract, store the creation address in the receipt.
		// 如果事务创建了一个合约，就将这个合约地址存储在收据中
		if msg.To() == nil {
			receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce())
		}
		// Set the receipt logs and create a bloom for filtering
		// 设置收据的布隆过滤器
		receipt.Logs = statedb.GetLogs(tx.Hash())
		receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
	
		return receipt, gas, err
	}