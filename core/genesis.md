# genesis #

----------

SetupGenesisBlock在db中写入或更新genesis块。

将使用的块是：

<table>
<thead>
<tr>
<th>----</th>
<th>genesis == nil</th>
<th>genesis != nil</th>
</tr>
</thead>
<tbody>
<tr>
<td>db没有创世块</td>
<td>主网网默认的创世块</td>
<td>当前给出的genesis块</td>
</tr>
<tr>
<td>db有创世块</td>
<td>使用数据库的创世块</td>
<td>当前给出的genesis块如果兼容</td>

</tr>
</tbody>
</table>

SetupGenesisBlock返回的链配置永远不会为空。

	func SetupGenesisBlock(db ethdb.Database, genesis *Genesis) (*params.ChainConfig, common.Hash, error) {
		//  创世块为配置
		if genesis != nil && genesis.Config == nil {
			return params.AllEthashProtocolChanges, common.Hash{}, errGenesisNoConfig
		}
	
		// Just commit the new block if there is no stored genesis block.
		// 如果本地没有存储创世块
		stored := rawdb.ReadCanonicalHash(db, 0)
		if (stored == common.Hash{}) { //本地没有保存创世块
			if genesis == nil {
				log.Info("Writing default main-net genesis block")
				// 使用主网默认的创世块
				genesis = DefaultGenesisBlock()
			} else {
				log.Info("Writing custom genesis block")
			}
			// 存储当前给出的genesis，并返回它的配置
			block, err := genesis.Commit(db)
			return genesis.Config, block.Hash(), err
		}
	
		// Check whether the genesis block is already written.
		if genesis != nil {
			hash := genesis.ToBlock(nil).Hash()
			if hash != stored { //与本地的创世块不匹配
				return genesis.Config, hash, &GenesisMismatchError{stored, hash}
			}
		}
	
		// Get the existing chain configuration.
		newcfg := genesis.configOrDefault(stored)
		storedcfg := rawdb.ReadChainConfig(db, stored)
		if storedcfg == nil {
			log.Warn("Found genesis block without chain config")
			rawdb.WriteChainConfig(db, stored, newcfg)
			return newcfg, stored, nil
		}
		// Special case: don't change the existing config of a non-mainnet chain if no new
		// config is supplied. These chains would get AllProtocolChanges (and a compat error)
		// if we just continued here.
		if genesis == nil && stored != params.MainnetGenesisHash {
			return storedcfg, stored, nil
		}
	
		// Check config compatibility and write the config. Compatibility errors
		// are returned to the caller unless we're already at block zero.
		// 检查配置兼容性并编写配置。 兼容性错误将返回给调用者，除非我们已经处于块0。
		height := rawdb.ReadHeaderNumber(db, rawdb.ReadHeadHeaderHash(db))
		if height == nil {
			return newcfg, stored, fmt.Errorf("missing block number for head header hash")
		}
		compatErr := storedcfg.CheckCompatible(newcfg, *height)
		if compatErr != nil && *height != 0 && compatErr.RewindTo != 0 {
			return newcfg, stored, compatErr
		}
		rawdb.WriteChainConfig(db, stored, newcfg)
		return newcfg, stored, nil
	}