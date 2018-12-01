# config #

----------
ChainConfig是确定区块链设置的核心配置。

ChainConfig基于每个块存储在数据库中。 这意味着由其创世块标识的任何网络都可以拥有自己的一组配置选项。
	
	type ChainConfig struct {
		//标识当前链并用于重放保护
		ChainID *big.Int `json:"chainId"` // chainId identifies the current chain and is used for replay protection
		//主网络的第1150000个区块，进入第二个阶段
		HomesteadBlock *big.Int `json:"homesteadBlock,omitempty"` // Homestead switch block (nil = no fork, 0 = already homestead)
		//DAO分叉，第1920000高度的块
		DAOForkBlock   *big.Int `json:"daoForkBlock,omitempty"`   // TheDAO hard-fork switch block (nil = no fork)
		DAOForkSupport bool     `json:"daoForkSupport,omitempty"` // Whether the nodes supports or opposes the DAO hard-fork
	
		// EIP150 implements the Gas price changes (https://github.com/ethereum/EIPs/issues/150)
		//EIP150实施gas价格变动，在2463000高度
		EIP150Block *big.Int    `json:"eip150Block,omitempty"` // EIP150 HF block (nil = no fork)
		EIP150Hash  common.Hash `json:"eip150Hash,omitempty"`  // EIP150 HF hash (needed for header only clients as only gas pricing changed)
		//简单重放攻击的保护，在2675000高度
		EIP155Block *big.Int `json:"eip155Block,omitempty"` // EIP155 HF block
		EIP158Block *big.Int `json:"eip158Block,omitempty"` // EIP158 HF block
		//Byzantium分叉高度4370000
		ByzantiumBlock      *big.Int `json:"byzantiumBlock,omitempty"`      // Byzantium switch block (nil = no fork, 0 = already on byzantium)
		//高度4230000
		ConstantinopleBlock *big.Int `json:"constantinopleBlock,omitempty"` // Constantinople switch block (nil = no fork, 0 = already activated)
		EWASMBlock          *big.Int `json:"ewasmBlock,omitempty"`          // EWASM switch block (nil = no fork, 0 = already activated)
	
		// Various consensus engines
		// 共识引擎
		Ethash *EthashConfig `json:"ethash,omitempty"`
		Clique *CliqueConfig `json:"clique,omitempty"`
	}

以太坊分为4各阶段：

1. Frontier  //边境
1. Homestead //家园
1. Metropolis //都会
1. Serenity   //宁静


## 以太坊硬分叉 ##

**Block #0**

"Frontier"
 - 以太坊的初始阶段，持续时间为2015年7月30日至2016年3月。

**Block #200,000**

"Ice Age" - 引入指数难度增加的硬分叉，促使向 Proof-of-Stake 过渡。

**Block #1,150,000**

"Homestead" - 以太坊的第二阶段，于2016年3月推出。


- Homestead是以太坊平台的第二个阶段，包括几个协议改变（protocol changes）和一个网络改变（networking change），这些变化使得我们能够对网络做进一步升级。
- EIP-2 主要的Homestead硬分叉改变（EIP是指以太坊改进提议（Ethereal Improvement Proposal））
- EIP-7 硬分叉相对应的EVM（以太坊虚拟机）更新：DELEGATECALL
- EIP-8 devp2p 向前兼容性


**Block #1,192,000**

"DAO" - 扭转了被攻击的DAO合约并导致以太坊和以太坊经典分裂成两个竞争系统的硬分叉。


**Block #2,463,000**

"Tangerine Whistle" - 改变某些IO运算的gas计算，并从拒绝服务攻击中清除累积状态，该攻击利用了这些操作的低gas成本。也就是配置中的EIP150Block

**Block #2,675,000**

"Spurious Dragon" - 一个解决更多拒绝服务攻击媒介的硬分叉，以及另一种状态清除。此外，还有重播攻击保护机制。也就是配置中的EIP155Block

**Block #4,370,000**

"Metropolis Byzantium" - Metropolis是以太坊的第三个阶段，目前在撰写本书时，于2017年10月推出.Byzantium是Metropolis的两个硬分叉中的第一个。为之后的Constantinople硬分叉做好了铺垫，共识机制逐步会从Proof of Work转换成Proof of Stake，挖矿奖励从5降至3，叔块奖励减少，加入EVM新指令，新函数。

**Constantinople**

Metropolis阶段的第二部分，计划于2018年中期。预计将包括切换到混合POW/POS共识算法，以及其他变更。
10月19日星期五的会议上，以太坊（ETH）的核心开发人员已达成共识，推迟计划的Constantinople协议硬分叉至2019年1月。

----------



**以太坊难度炸弹**


[原文链接](http://www.qukuaiwang.com.cn/news/11490.html)

**难度炸弹**：以太坊的“难度炸弹”（“Difficulty Bomb”）指的是，在挖掘算法中，使用以太币在区块链上对矿工进行奖励的难度越来越大。随着游戏变得更加复杂（矿工发现以太币难挣得多），在以太坊区块链上块的生产之间将会有相当长的时间差。这将以指数的方式放缓，其对矿商的吸引力也将下降。这个场景的开始被称为“以太坊冰期”。在这段时间里，以太坊将从“工作证明”（PoW）过渡到“利益证明”（PoS），PoW要求矿工通过相互竞争来解决难题并赚取回报，而PoS则根据投资或硬币所有权来分配奖励。作为以太坊的Casper更新的一部分，协议之间的切换将在今年晚些时候进行。在这段时间内难度炸弹将阻止以太坊区块链分叉。

为什么要引入难度炸弹？

以太坊的难度炸弹对矿工来说是一种威慑，即使在区块链过渡到PoS后，他们可能会选择继续使用PoW。他们这样做的主要原因可能是权力(power)和利润的天平从矿主转移到了投资者和区块链使用者手中。如果不是所有的矿工都转换到PoS，那么以太坊区块链可能就面临着分叉的危险。2017年也出现了类似的情况，当时比特币的矿商们通过支持比特币现金，迫使其成为区块链的一个分支。然而，以太坊的创始人预见到了这种可能性，并对它的块链进行了编程，增加了挖掘算法的难度。

----------
