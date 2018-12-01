# state_transition #

----------

状态转换模型

状态转换 是指用当前的world state来执行交易，并改变当前的world state

状态转换做了所有所需的工作来产生一个新的有效的state root

1) Nonce handling  Nonce 处理

2) Pre pay gas     预先支付Gas

3) Create a new state object if the recipient is \0*32 如果接收人是空，那么创建一个新的state object

4) Value transfer  转账

== If contract creation ==

  4a) Attempt to run transaction data 尝试运行输入的数据

  4b) If valid, use result as code for the new state object 如果有效，那么用运行的结果作为新的state object的code

== end ==

5) Run Script section 运行脚本部分

6) Derive new state root 导出新的state root


----------


StateTransition 结构体

	type StateTransition struct {
		gp         *GasPool		//用来追踪区块内部的Gas的使用情况
		msg        Message		// Message Call
		gas        uint64
		gasPrice   *big.Int		// gas的价格
		initialGas uint64		// 最开始的gas
		value      *big.Int		// 转账的值
		data       []byte		// 数据
		state      vm.StateDB	// 状态数据库StateDB
		evm        *vm.EVM		// 虚拟机
	}

Message 是合约之间调用得消息

	type Message interface {
		From() common.Address  //发送者
		//FromFrontier() (common.Address, error)
		To() *common.Address   //接收者
	
		GasPrice() *big.Int		//gas价格
		Gas() uint64			//message 的 GasLimit
		Value() *big.Int
	
		Nonce() uint64
		CheckNonce() bool
		Data() []byte
	}

IntrinsicGas 

使用给定数据计算消息的“固有气体”。
	
	func IntrinsicGas(data []byte, contractCreation, homestead bool) (uint64, error) {
		// Set the starting gas for the raw transaction
		var gas uint64
		// 首先是基本gas消耗
		if contractCreation && homestead {
			gas = params.TxGasContractCreation  //创建合约需要消耗得gas
		} else {
			gas = params.TxGas		//普通事务所需要得gas
		}
		// Bump the required gas by the amount of transactional data
		// 通过交易数据量来压缩所需的气体
		if len(data) > 0 {
			// Zero and non-zero bytes are priced differently
			var nz uint64
			for _, byt := range data {
				if byt != 0 {
					nz++
				}
			}
			// Make sure we don't exceed uint64 for all data combinations
			// 确保所有数据组合都不超过uint64
			// 也就是说除去基本消耗后，还能够满足多少字姐gas消耗，hz需要不高于这个值，否则将会透支
			// 例如如果是普通事务nz不能超过271275648142787214字节
			if (math.MaxUint64-gas)/params.TxDataNonZeroGas < nz {
				return 0, vm.ErrOutOfGas
			}
			gas += nz * params.TxDataNonZeroGas
	
			z := uint64(len(data)) - nz //空字节数z
			if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
				return 0, vm.ErrOutOfGas
			}
			//再加上空字节数
			gas += z * params.TxDataZeroGas
		}
		return gas, nil
	}

NewStateTransition 初始化新的StateTransition并返回

	func NewStateTransition(evm *vm.EVM, msg Message, gp *GasPool) *StateTransition {
		return &StateTransition{
			gp:       gp,
			evm:      evm,
			msg:      msg,
			gasPrice: msg.GasPrice(),
			value:    msg.Value(),
			data:     msg.Data(),
			state:    evm.StateDB,
		}
	}


ApplyMessage通过对环境中的旧状态应用给定消息来计算新状态。

ApplyMessage返回任何EVM执行（如果发生）返回的字节，使用的气体（包括气体退款）和失败时的错误。 错误始终表示核心错误，这意味着该消息对于该特定状态始终会失败，并且永远不会在块中被接受。

	func ApplyMessage(evm *vm.EVM, msg Message, gp *GasPool) ([]byte, uint64, bool, error) {
		return NewStateTransition(evm, msg, gp).TransitionDb()
	}


TransitionDb()将通过应用当前消息并返回包括使用过的气体的结果来转换状态。 如果失败则返回错误。错误表示存在共识问题。

	func (st *StateTransition) TransitionDb() (ret []byte, usedGas uint64, failed bool, err error) {
			// 检查nonce,判断余额等
			if err = st.preCheck(); err != nil {
			return
			}
			msg := st.msg
			//设置发送方
			sender := vm.AccountRef(msg.From())
			// 判断分叉，是否是homestead阶段
			homestead := st.evm.ChainConfig().IsHomestead(st.evm.BlockNumber)
			// 判断是否是创建合约
			contractCreation := msg.To() == nil
	
			// Pay intrinsic gas
			// 支付固有气体
			gas, err := IntrinsicGas(st.data, contractCreation, homestead)
			if err != nil {
			return nil, 0, false, err
			}
			// 将事务原有的gas余额减去需要支付的固有gas消耗
			if err = st.useGas(gas); err != nil {
			return nil, 0, false, err
		}
	
		var (
			evm = st.evm
			// vm errors do not effect consensus and are therefor
			// not assigned to err, except for insufficient balance
			// error.
			// vm错误不会影响共识，因此不会分配给错误，除非余额不足。
			vmerr error
		)
		if contractCreation {
			// 创建合约
			ret, _, st.gas, vmerr = evm.Create(sender, st.data, st.gas, st.value)
		} else {
			// Increment the nonce for the next transaction
			// 增加Nonce
			st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
			// 运行事务
			ret, st.gas, vmerr = evm.Call(sender, st.to(), st.data, st.gas, st.value)
		}
		if vmerr != nil {
			log.Debug("VM returned with error", "err", vmerr)
			// The only possible consensus-error would be if there wasn't
			// sufficient balance to make the transfer happen. The first
			// balance transfer may never fail.
			if vmerr == vm.ErrInsufficientBalance {
				return nil, 0, false, vmerr
			}
		}
		// 返还剩余的气体，余额
		st.refundGas()
		//支付手续费
		st.state.AddBalance(st.evm.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.gasPrice))
	
		return ret, st.gasUsed(), vmerr != nil, err
	}

preCheck 检查nonce的值

	func (st *StateTransition) preCheck() error {
		// Make sure this transaction's nonce is correct.
		if st.msg.CheckNonce() {
			nonce := st.state.GetNonce(st.msg.From())
			if nonce < st.msg.Nonce() {
				return ErrNonceTooHigh
			} else if nonce > st.msg.Nonce() {
				return ErrNonceTooLow
			}
		}
		return st.buyGas()
	}
	
	func (st *StateTransition) buyGas() error {
		mgval := new(big.Int).Mul(new(big.Int).SetUint64(st.msg.Gas()), st.gasPrice) //需要花费的价钱
	
		// 先判断余额，再判断气体
		if st.state.GetBalance(st.msg.From()).Cmp(mgval) < 0 { //余额不足无法支付gas
			return errInsufficientBalanceForGas
		}
		// 块中扣除气体
		if err := st.gp.SubGas(st.msg.Gas()); err != nil {
			return err
		}
		st.gas += st.msg.Gas()
	
		st.initialGas = st.msg.Gas()
		// 扣除相应的余额
		st.state.SubBalance(st.msg.From(), mgval)
		return nil
	}