# hyperledger fabric中区块打包的流程

- [hlf引用的goleveldb的lru算法](vendor/github.com/syndtr/goleveldb/leveldb/cache/lru.go)



- core/ledger/kvledger/kv_ledger.go
```go
func (l *kvLedger) CommitPvtDataOfOldBlocks(reconciledPvtdata []*ledger.ReconciledPvtdata) ([]*ledger.PvtdataHashMismatch, error) {
	logger.Debugf("[%s:] Comparing pvtData of [%d] old blocks against the hashes in transaction's rwset to find valid and invalid data",
		l.ledgerID, len(reconciledPvtdata))

	hashVerifiedPvtData, hashMismatches, err := constructValidAndInvalidPvtData(reconciledPvtdata, l.blockStore)
	if err != nil {
		return nil, err
	}

	err = l.applyValidTxPvtDataOfOldBlocks(hashVerifiedPvtData)
	if err != nil {
		return nil, err
	}

	logger.Debugf("[%s:] Committing pvtData of [%d] old blocks to the pvtdatastore", l.ledgerID, len(reconciledPvtdata))
	err = l.blockStore.CommitPvtDataOfOldBlocks(hashVerifiedPvtData)
	if err != nil {
		return nil, err
	}

	return hashMismatches, nil
}
```


- core/ledger/ledgerstorage/store.go
```go
// CommitWithPvtData commits the block and the corresponding pvt data in an atomic operation
func (s *Store) CommitWithPvtData(blockAndPvtdata *ledger.BlockAndPvtData) error {
	blockNum := blockAndPvtdata.Block.Header.Number
	s.rwlock.Lock()
	defer s.rwlock.Unlock()

	pvtBlkStoreHt, err := s.pvtdataStore.LastCommittedBlockHeight()
	if err != nil {
		return err
	}

	if pvtBlkStoreHt < blockNum+1 { // The pvt data store sanity check does not allow rewriting the pvt data.
		// when re-processing blocks (rejoin the channel or re-fetching last few block),
		// skip the pvt data commit to the pvtdata blockstore
		logger.Debugf("Writing block [%d] to pvt block store", blockNum)
		// If a state fork occurs during a regular block commit,
		// we have a mechanism to drop all blocks followed by refetching of blocks
		// and re-processing them. In the current way of doing this, we only drop
		// the block files (and related artifacts) but we do not drop/overwrite the
		// pvtdatastorage as it might leads to data loss.
		// During block reprocessing, as there is a possibility of an invalid pvtdata
		// transaction to become valid, we store the pvtdata of invalid transactions
		// too in the pvtdataStore as we do for the publicdata in the case of blockStore.
		pvtData, missingPvtData := constructPvtDataAndMissingData(blockAndPvtdata)
		if err := s.pvtdataStore.Commit(blockAndPvtdata.Block.Header.Number, pvtData, missingPvtData); err != nil {
			return err
		}
	} else {
		logger.Debugf("Skipping writing block [%d] to pvt block store as the store height is [%d]", blockNum, pvtBlkStoreHt)
	}

	if err := s.AddBlock(blockAndPvtdata.Block); err != nil {
		return err
	}

	if pvtBlkStoreHt == blockNum+1 {
		// we reach here only when the pvtdataStore was ahead
		// of blockStore during the store opening time (would
		// occur after a peer rollback/reset).
		s.isPvtstoreAheadOfBlockstore.Store(false)
	}

	return nil
}
```


- core/ledger/pvtdatastorage/store_impl.go
```go
// Prepare implements the function in the interface `Store`
func (s *store) Commit(blockNum uint64, pvtData []*ledger.TxPvtData, missingPvtData ledger.TxMissingPvtDataMap) error {
	expectedBlockNum := s.nextBlockNum()
	if expectedBlockNum != blockNum {
		return &ErrIllegalArgs{fmt.Sprintf("Expected block number=%d, received block number=%d", expectedBlockNum, blockNum)}
	}

	batch := leveldbhelper.NewUpdateBatch()
	var err error
	var keyBytes, valBytes []byte

	storeEntries, err := prepareStoreEntries(blockNum, pvtData, s.btlPolicy, missingPvtData)
	if err != nil {
		return err
	}

	for _, dataEntry := range storeEntries.dataEntries {
		keyBytes = encodeDataKey(dataEntry.key)
		if valBytes, err = encodeDataValue(dataEntry.value); err != nil {
			return err
		}
		batch.Put(keyBytes, valBytes)
	}

	for _, expiryEntry := range storeEntries.expiryEntries {
		keyBytes = encodeExpiryKey(expiryEntry.key)
		if valBytes, err = encodeExpiryValue(expiryEntry.value); err != nil {
			return err
		}
		batch.Put(keyBytes, valBytes)
	}

	for missingDataKey, missingDataValue := range storeEntries.missingDataEntries {
		keyBytes = encodeMissingDataKey(&missingDataKey)
		if valBytes, err = encodeMissingDataValue(missingDataValue); err != nil {
			return err
		}
		batch.Put(keyBytes, valBytes)
	}

	committingBlockNum := s.nextBlockNum()
	logger.Debugf("Committing private data for block [%d]", committingBlockNum)
	batch.Put(lastCommittedBlkkey, encodeLastCommittedBlockVal(committingBlockNum))
	if err := s.db.WriteBatch(batch, true); err != nil {
		return err
	}

	s.isEmpty = false
	atomic.StoreUint64(&s.lastCommittedBlock, committingBlockNum)
	logger.Debugf("Committed private data for block [%d]", committingBlockNum)
	s.performPurgeIfScheduled(committingBlockNum)
	return nil
}
```