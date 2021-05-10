---
title: CDC_Architecture_Analysis
author: shimengfei
avatar: 'https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/lover.jpg'
authorLink: shimengfei.github.io
authorAbout: stone
authorDesc: stone
categories: 技术
comments: true
photos: 'https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/lover.jpg'
abbrlink: 32752
date: 2021-03-08 20:45:35
tags:
keywords:
description:
cover: https://cdn.jsdelivr.net/gh/shimengfei/cdn/guidao/8.png
---

## CDC整体架构分析
### CDC 概览
![cdc-overview.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/cdc-overview.png)
### CDC流程图
![cdc-seq.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/cdc-seq.png)
***

### Range Feed原理分析
**RangeFeed数据来源**
下图中的步骤③当raft group leader 将raft log apply到状态机时，将raftCommand中对应的数据变更publish出来
![raft.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/raft.png)
**RangeFeed捕获变更数据流程**
![rangefeed.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/rangefeed.png)
**Feed结构分析**
![cdc ca.png.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/cdc ca.png)

***
#### The Difficulty With Decoding WriteBatches

**MVCC操作对应的WriteBatch包含的entries**

```go
- MVCCPut
    - inline (never transactional)
        + write meta key w/ inline value
    - not inline
        - transactional
            - no
                + write new version
            - yes
                - intent exists with older timestamp
                    + clear old version
                    + write meta
                    + write new version
                - else
                    + write meta
                    + write new version
- MVCCDelete
    - inline (never transactional)
        + clear meta key
    - not inline
        + same as MVCCPut but with empty value and Deleted=true in meta
- MVCCDeleteRange
    + calls MVCCDelete for each key in range
- MVCCMerge
    + write merge record (never transactional)
- MVCCResolveWriteIntent
    - commit
        + clear meta
        - if timestamp of intent is incorrect
            + clear old version key
            + write new version key
    - abort
        + clear version key
        + clear meta key
    - push
        + write meta with larger timestamp
        + clear old version key
        + write new version key
- MVCCGarbageCollect
    + clear each meta key
    + clear each version
```
解码 ***WriteBatches*** 的过程将包括获取RocksDB批处理条目列表并对其正在执行的集体更高级别的操作进行反向工程。例如，包含对同一个逻辑键的meta键和版本键的写操作的WriteBatch将被解释为一个意向写操作;同样，删除meta键将被解释为一个成功的intent解析操作。这里不仅需要进行大量的状态转换以进行模式匹配，而且还不清楚在没有额外的引擎读取的情况下这种解码是否会变得清晰无误，也就是说，对WriteBatch本身进行解码的过程是任何方式都是无法克服的。更令人担忧的是，这正在对筏上MVCC层的实现细节造成非常严重的筏下依赖。引入这种依赖性后，我们将必须非常谨慎地对MVCC进行任何更改，这可能会导致严重的后果。诸如此类的问题促使提议者评估kv重构，因此不应掉以轻心。

#### Logical MVCC Operations

因此，我们提出了一种替代方法来对`WriteBatch`进行解码直接在Raft命令中。相反，我们建议引入一种逻辑每个Raft命令的更高级MVCC操作的列表。这些更高层次的操作将不受物理操作语义的约束描述在`WriteBatch`中，因此不会限制对其的更改将来。取而代之的是，这些操作将描述批次的更改在MVCC级别。这样可以避免以前的问题，并且更容易解码Raft下游的操作。

***LogicalOps***
`RaftCommand` proto message:
```
repeated MVICLogicalOp mvcc_ops;
```

`MVICLogicalOp`定义:
``` go
message Bytes {
    option (gogoproto.onlyone) = true;

    bytes inline = 1;

    message Pointer {
        int32 offset = 1;
        int32 len = 2;
    }
    Pointer pointer = 2;
}

message MVCCWriteValueOp {
    bytes key = 1;
    util.hlc.Timestamp timestamp = 2;
    bytes value = 3;
}

message MVCCWriteIntentOp {
    bytes txn_id = 1;
    bytes txn_key = 2;
    util.hlc.Timestamp timestamp = 3;
}

message MVCCUpdateIntentOp {
    bytes txn_id = 1;
    util.hlc.Timestamp timestamp = 2;
}

message MVCCCommitIntentOp {
    bytes txn_id = 1;
    bytes key = 2;
    util.hlc.Timestamp timestamp = 3;
    bytes value = 4;
}

message MVCCAbortIntentOp {
    bytes txn_id = 1;
}

message MVICLogicalOp {
    option (gogoproto.onlyone) = true;

    MVCCWriteValueOp   write_value   = 1;
    MVCCWriteIntentOp  write_intent  = 2;
    MVCCUpdateIntentOp update_intent = 3;
    MVCCCommitIntentOp commit_intent = 4;
    MVCCAbortIntentOp  abort_intent  = 5;
}
```

值得注意的是，所有字节切片值将可选地指向WriteBatch本身，这将有助于限制由于复制物理和逻辑操作而导致的写放大。 此优化可以逐步引入。 我们还可以探索压缩技术，这可能会导致类似的重复数据删除。在Raft之上，随着在BatchRequest中处理每个请求，MVCCOps的日志将与WriteBatch并排构造。 将Raft及其MVCC操作分开，然后分别通过以下逻辑运行它们：
``` go
for op in mvccOps {
    match op {
        WriteValue(key, ts, val) => SendValueToMatchingFeeds(key, ts, val),
        WriteIntent(txnID, ts)   => {
            UnresolvedIntentQueue[txnID].refcount++
            UnresolvedIntentQueue[txnID].Timestamp.Forward(ts)
        }, 
        UpdateIntent(txnID, ts) => UnresolvedIntentQueue[txnID].Timestamp.Forward(ts),
        CommitIntent(txnID, key, ts) => {
            UnresolvedIntentQueue[txnID].refcount--
            // It's unfortunate that we'll need to perform an engine lookup
            // for this, but it's not a huge deal. The value should almost
            // certainly be in RocksDB's memtable or block cache, so there's
            // not much of a reason for any extra layer of caching.
            SendValueToMatchingFeeds(key, MVCCGet(key, ts), ts)
        },
        AbortIntent    => UnresolvedIntentQueue[txnID].refcount--,
    }
}
```


## CDC 代码时序图编写


## CDC最终一致性


建立CockroachDB changefeed的最大挑战从一开始就很明显。我们希望我们的changefeed可以横向扩展，同时我们也希望它们保持强大的事务语义。

在单节点数据库中，比如MySQL, 它维护一个binlog记录数据的变更,因此构建changefeed的工作主要是以合理的方式公开此日志.其他的数据库也类似。但是，CockroachDB具有独特的分布式架构。它存储的数据被分解为大约64MB的“ranges”。这些ranges每个都被复制成N个“副本”以获得高可用。CockroachDB事务可以涉及任何或所有这些ranges，这意味着它可以跨越集群中的任何或所有节点。这与在水平扩展其他SQL数据库时使用的“分片”设置形成对比，其中每个分片是完全独立的复制单元，并且事务不能跨越分片。然后，分片SQL群集上的changefeed只是每个分片的changefeed，通常由分片的领导者运行,如下图所示。由于每个事务完全发生在一个分片中，因此分片之间事务的相对排序并不那么值得特别关注（或者说大家很多时候不在乎这种分片之间的事务排序）。它还意味着各个分片的feeds可以完全并行化（每个分片一个Kafka主题或分区是典型的）。
![range_database.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/range_database.png)
图1：分片SQL数据库中的事务无法跨越分片。

由于CockroachDB事务可以使用集群中的任何range集合（考虑跨分片事务），因此事务排序要复杂得多, 如下图所示。特别是，并不总是可以将事务划分为独立的流。这里简单的答案是将每个事务放入一个流中，但我们对此并不满意。CockroachDB旨在水平扩展到大量节点，因此我们当然希望我们的changefeed也可以水平扩展。
![znbase_txn.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/znbase_txn.png)

图2：CockroachDB中的事务可以跨节点。

### 分布式事务保证事务语义
CockroachDB中的SQL表可以跨越多个range，但该表中的每一行始终包含在一个range内。（当range变大时系统可以移动，系统将其分成两部分以及range变小并且系统将其合并到相邻range时，但这些可以单独处理。）此外，每个range都是一个独立的raft group，因此有自己的WAL，我们可以追随这个WAL。这意味着我们可以为每个SQL行生成有序的changefeed。为了实现这一目标，我们开发了一种内部机制RangeFeed，将这些变化直接从我们的raft group中推出，而不是轮询它们。

每个Row流都是独立的，这意味着我们可以水平缩放它们。使用我们的分布式SQL框架，我们将处理器放置在正在观察的数据旁边发出行更改，从而消除不必要的网络跃点。如果一个节点完成所有观看和发送，它还可以避免我们遇到的单点故障,如下图所示。
![znbase_cdc.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/znbase_cdc.png)
图3：CockroachDB range leader各自直接向Kafka（或其他接收器）发出changefeed。

对于许多changefeed用途，这就足够了,每条消息都可以触发移动推送通知，某些数据存储不支持事务。有序的行流对这两者都很有用。对于其他用途，这还不够; 将数据镜像到分析数据库当然不希望应用部分事务。

每个CockroachDB事务使用相同的HLC时间戳提交每一行。在每个消息中为更改的行暴露此时间戳就足以获得事务信息（按时间戳分组行集合）[1]以及总排序（按时间戳排序行）。在我们现有的事务时间戳之上构建意味着我们的changefeed与CockroachDB中的其他所有内容具有相同的可序列化保证。

那最后的问题是知道何时进行这一组或排序。如果hlc1从一个CockroachDB节点随时间发出更改的行，那么在对其进行操作之前，您需要等待多长时间才能确保其他任何节点都没有更改hlc1？

我们用一个我们称之为“resolved”的时间戳消息的概念来解决这个问题。这是一个承诺，即不会发出新的行更改，其时间戳小于或等于已解析的时间戳消息中的时间戳。这意味着上述用户可以在hlc1从每个节点[2]接收到已解决的时间戳之后进行操作>= hlc1, 如下图所示。
![cdc resolved.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/cdc resolved.png)

图4中为事务发出的前几条消息的一种可能排序。

在图4中，想象两个独立的流都已经通过读取X。hlc1在其中一个stream上已经“resolved”，但在另外一个stream上没有“resolved”，所以hlc1还没有“resolved。

现在想象一下，在稍后的某些时候，消息已经被读过了Y。两个stream都已“resolved”了hlc1，所以我们知道我们已收到所有已发生的变化，包括hlc1。如果我们按时间戳对消息进行分组，我们可以恢复交易。在这种情况下，只有(B->1,C->2)，承诺在hlc1。此事务现在可以发送到分析数据库。

请注意，(A->3)更改发生在hlc2，因此尚未“resolved”。这意味着changefeed用户需要继续缓存它。

我们还可以随时重建数据库的状态，包括hlc1保持每行的最新值。这甚至适用于范围和节点。在这种情况下，hlc1数据库时B=1,C=2。

最后，想象一下稍后Z读取所有消息的时间。通过相同的两个进程再次获取数据库的事务和状态。在这种情况下，交易(A->3,B->4)承诺hlc2和(C->5)承诺hlc3。在hlc3包含的数据库中A=3,B=1,C=5。请注意，hlc2如有必要，我们还可以重建数据库。

这个创新的解决方案来源于一篇论文《Naiad: A Timely Dataflow System》,其核心思想类似于TCP的接收时间窗口,CockroachDB就是基于这样的思想设计自己的CDC解决方案.

#### 现存问题:
**自动启动CDC:CockroachDB的CDC是表级别的,但是对于一般的同步任务来说需要至少在DB层面进行配置,以便简化处理,这里的挑战是对于大部分消息队列,只能保证单分片上的数据有序性,而我们在table层面需要这种有序性才能正确的将数据变更按照事务粒度重新组织。**

**事务捕获:CockroachDB的CDC并不是事务粒度,而是行级别分散的,另外CockroachDB不相干的事务的时间顺序没有严格要求,导致从消息队列中消费的事件的时间戳是混乱的,并且可能存在重复的事件,因此首先我们需要在消费端按照事务作为最小单元重新整理这些数据,这个的主要依据就是事件携带的时间戳,相同时间戳的事件被认为是属于同一个事务的(两个不重叠的事务可以使用相同的时间戳进行提交，但它们具有纳秒精度，因此在实践中这种情况很少见).然后我们按照时间大小顺序重新排序,其中需要对重复的数据进行过滤.**

***
## 代码研读

```go
// stageRaftCommand handles the first phase of applying a command to the
// replica state machine.
//
// The proposal also contains auxiliary data which needs to be verified in order
// to decide whether the proposal should be applied: the command's MaxLeaseIndex
// must move the state machine's LeaseAppliedIndex forward, and the proposer's
// lease (or rather its sequence number) must match that of the state machine,
// and lastly the GCThreshold is validated. If any of the checks fail, the
// proposal's content is wiped and we apply an empty log entry instead. If an
// error occurs and the command was proposed locally, the error will be
// communicated to the waiting proposer. The two typical cases in which errors
// occur are lease mismatch (in which case the caller tries to send the command
// to the actual leaseholder) and violation of the LeaseAppliedIndex (in which
// case the proposal is retried if it was proposed locally).
//
// Assuming all checks were passed, the command is applied to the batch,
// which is done by the aptly named applyRaftCommandToBatch.
//
// For trivial proposals this is the whole story, but some commands trigger
// additional code in this method in this method via a side effect (in the
// proposal's ReplicatedEvalResult or, for local proposals,
// LocalEvalResult). These might, for example, trigger an update of the
// Replica's in-memory state to match updates to the on-disk state, or pass
// intents to the intent resolver. Some commands don't fit this simple schema
// and need to hook deeper into the code. Notably splits and merges need to
// acquire locks on their right-hand side Replicas and may need to add data to
// the WriteBatch before it is applied; similarly, changes to the disk layout of
// internal state typically require a migration which shows up here. Any of this
// logic however is deferred until after the batch has been written to the
// storage engine.
func (r *Replica) stageRaftCommand(
	ctx context.Context,
	cmd *cmdAppCtx,
	batch engine.Batch,
	replicaState *storagepb.ReplicaState,
	writeAppliedState bool,
) {
	if cmd.e.Index == 0 {
		log.Fatalf(ctx, "processRaftCommand requires a non-zero index")
	}
	if log.V(4) {
		log.Infof(ctx, "processing command %x: maxLeaseIndex=%d",
			cmd.idKey, cmd.raftCmd.MaxLeaseIndex)
	}

	var ts hlc.Timestamp
	if cmd.idKey != "" {
		ts = cmd.replicatedResult().Timestamp
	}

	cmd.leaseIndex, cmd.proposalRetry, cmd.forcedErr = checkForcedErr(ctx,
		cmd.idKey, cmd.raftCmd, cmd.proposal, cmd.proposedLocally(), replicaState)

	// applyRaftCommandToBatch will return "expected" errors, but may also indicate
	// replica corruption (as of now, signaled by a replicaCorruptionError).
	// We feed its return through maybeSetCorrupt to act when that happens.
	if cmd.forcedErr != nil {
		log.VEventf(ctx, 1, "applying command with forced error: %s", cmd.forcedErr)
	} else {
		log.Event(ctx, "applying command")

		if splitMergeUnlock, err := r.maybeAcquireSplitMergeLock(ctx, cmd.raftCmd); err != nil {
			log.Eventf(ctx, "unable to acquire split lock: %s", err)
			// Send a crash report because a former bug in the error handling might have
			// been the root cause of #19172.
			_ = r.store.stopper.RunAsyncTask(ctx, "crash report", func(ctx context.Context) {
				log.SendCrashReport(
					ctx,
					&r.store.cfg.Settings.SV,
					0, // depth
					"while acquiring split lock: %s",
					[]interface{}{err},
					log.ReportTypeError,
				)
			})

			cmd.forcedErr = roachpb.NewError(err)
		} else if splitMergeUnlock != nil {
			// Set the splitMergeUnlock on the cmdAppCtx to be called after the batch
			// has been applied (see applyBatch).
			cmd.splitMergeUnlock = splitMergeUnlock
		}
	}

	if filter := r.store.cfg.TestingKnobs.TestingApplyFilter; cmd.forcedErr == nil && filter != nil {
		var newPropRetry int
		newPropRetry, cmd.forcedErr = filter(storagebase.ApplyFilterArgs{
			CmdID:                cmd.idKey,
			ReplicatedEvalResult: *cmd.replicatedResult(),
			StoreID:              r.store.StoreID(),
			RangeID:              r.RangeID,
		})
		if cmd.proposalRetry == 0 {
			cmd.proposalRetry = proposalReevaluationReason(newPropRetry)
		}
	}

	if cmd.forcedErr != nil {
		// Apply an empty entry.
		*cmd.replicatedResult() = storagepb.ReplicatedEvalResult{}
		cmd.raftCmd.WriteBatch = nil
		cmd.raftCmd.LogicalOpLog = nil
	}

	// Run any triggers that should occur before the batch is applied
	// and before the write batch is staged in the batch.
	//获取Batch中存储的befor value,也即处理MVICLogicalOp针对于不同的Op进行相关处理，RangeFeed目前处理WriteValue Op以及CommitOp将其对应的Value publish
	if err := r.runPreApplyTriggersBeforeStagingWriteBatch(ctx, cmd, batch); err != nil {
		log.Errorf(ctx, "unable to update the state machine: %+v", err)
		log.Fatal(ctx, err)
	}
	// Update the node clock with the serviced request. This maintains
	// a high water mark for all ops serviced, so that received ops without
	// a timestamp specified are guaranteed one higher than any op already
	// executed for overlapping keys.
	// TODO(ajwerner): coalesce the clock update per batch.
	r.store.Clock().Update(ts)

	// If the command was using the deprecated version of the MVCCStats proto,
	// migrate it to the new version and clear out the field.
	if deprecatedDelta := cmd.replicatedResult().DeprecatedDelta; deprecatedDelta != nil {
		if cmd.replicatedResult().Delta != (enginepb.MVCCStatsDelta{}) {
			log.Fatalf(ctx, "stats delta not empty but deprecated delta provided: %+v", cmd)
		}
		cmd.replicatedResult().Delta = deprecatedDelta.ToStatsDelta()
		cmd.replicatedResult().DeprecatedDelta = nil
	}

	// Apply the Raft command to the batch's accumulated state. This may also
	// have the effect of mutating cmd.replicatedResult().
	err := r.applyRaftCommandToBatch(cmd.ctx, cmd, replicaState, batch, writeAppliedState)
	if err != nil {
		// applyRaftCommandToBatch returned an error, which usually indicates
		// either a serious logic bug in CockroachDB or a disk
		// corruption/out-of-space issue. Make sure that these fail with
		// descriptive message so that we can differentiate the root causes.
		log.Errorf(ctx, "unable to update the state machine: %+v", err)
		// Report the fatal error separately and only with the error, as that
		// triggers an optimization for which we directly report the error to
		// sentry (which in turn allows sentry to distinguish different error
		// types).
		log.Fatal(ctx, err)
	}

	// AddSSTable ingestions run before the actual batch gets written to the
	// storage engine. This makes sure that when the Raft command is applied,
	// the ingestion has definitely succeeded. Note that we have taken
	// precautions during command evaluation to avoid having mutations in the
	// WriteBatch that affect the SSTable. Not doing so could result in order
	// reversal (and missing values) here.
	//
	// NB: any command which has an AddSSTable is non-trivial and will be
	// applied in its own batch so it's not possible that any other commands
	// which precede this command can shadow writes from this SSTable.
	if cmd.replicatedResult().AddSSTable != nil {
		copied := addSSTablePreApply(
			ctx,
			r.store.cfg.Settings,
			r.store.engine,
			r.raftMu.sideloaded,
			cmd.e.Term,
			cmd.e.Index,
			*cmd.replicatedResult().AddSSTable,
			r.store.limiters.BulkIOWriteRate,
		)
		r.store.metrics.AddSSTableApplications.Inc(1)
		if copied {
			r.store.metrics.AddSSTableApplicationCopies.Inc(1)
		}
		cmd.replicatedResult().AddSSTable = nil
	}

	if cmd.replicatedResult().Split != nil {
		// Splits require a new HardState to be written to the new RHS
		// range (and this needs to be atomic with the main batch). This
		// cannot be constructed at evaluation time because it differs
		// on each replica (votes may have already been cast on the
		// uninitialized replica). Write this new hardstate to the batch too.
		// See https://github.com/cockroachdb/cockroach/issues/20629
		splitPreApply(ctx, batch, cmd.replicatedResult().Split.SplitTrigger)
	}

	if merge := cmd.replicatedResult().Merge; merge != nil {
		// Merges require the subsumed range to be atomically deleted when the
		// merge transaction commits.
		rhsRepl, err := r.store.GetReplica(merge.RightDesc.RangeID)
		if err != nil {
			log.Fatal(ctx, err)
		}
		const destroyData = false
		err = rhsRepl.preDestroyRaftMuLocked(ctx, batch, batch, merge.RightDesc.NextReplicaID, destroyData)
		if err != nil {
			log.Fatal(ctx, err)
		}
	}

	// Provide the command's corresponding logical operations to the Replica's
	// rangefeed. Only do so if the WriteBatch is non-nil, in which case the
	// rangefeed requires there to be a corresponding logical operation log or
	// it will shut down with an error. If the WriteBatch is nil then we expect
	// the logical operation log to also be nil. We don't want to trigger a
	// shutdown of the rangefeed in that situation, so we don't pass anything to
	// the rangefed. If no rangefeed is running at all, this call will be a noop.
	if cmd.raftCmd.WriteBatch != nil {
	//与之前的befer处理类似，	
	//获取Batch中存储的value,也即处理MVICLogicalOp针对于不同的Op进行相关处理，RangeFeed目前处理WriteValue Op以及CommitOp将其对应的Value publish
		r.handleLogicalOpLogRaftMuLocked(ctx, cmd.raftCmd.LogicalOpLog, batch)
	} else if cmd.raftCmd.LogicalOpLog != nil {
		log.Fatalf(ctx, "non-nil logical op log with nil write batch: %v", cmd.raftCmd)
	}
}



// runPreApplyTriggersBeforeStagingWriteBatch runs any triggers that must fire
// before a command is applied to the state machine but after the command is
// staged in the replicaAppBatch's write batch. It may modify the command.
func (r *Replica) runPreApplyTriggersBeforeStagingWriteBatch(
	ctx context.Context, cmd *cmdAppCtx, batch engine.Batch,
) error {
	if ops := cmd.raftCmd.LogicalOpLog; ops != nil {
		r.populatePrevValsInLogicalOpLogRaftMuLocked(ctx, ops, batch)
	}
	return nil
}




// populatePrevValsInLogicalOpLogRaftMuLocked updates the provided logical op
// log with previous values read from the reader, which is expected to reflect
// the state of the Replica before the operations in the logical op log are
// applied. No-op if a rangefeed is not active. Requires raftMu to be locked.
func (r *Replica) populatePrevValsInLogicalOpLogRaftMuLocked(
	ctx context.Context, ops *storagepb.LogicalOpLog, prevReader engine.Reader,
) {
	if r.raftMu.rangefeed == nil {
		return
	}
	if ops == nil {
		return
	}

	// Read from the Reader to populate the PrevValue fields.
	for _, op := range ops.Ops {
		var key []byte
		var ts hlc.Timestamp
		var prevValPtr *[]byte
		switch t := op.GetValue().(type) {
		case *enginepb.MVCCWriteValueOp:
			key, ts, prevValPtr = t.Key, t.Timestamp, &t.PrevValue
		case *enginepb.MVCCCommitIntentOp:
			key, ts, prevValPtr = t.Key, t.Timestamp, &t.PrevValue
		case *enginepb.MVCCWriteIntentOp,
			*enginepb.MVCCUpdateIntentOp,
			*enginepb.MVCCAbortIntentOp,
			*enginepb.MVCCAbortTxnOp:
			// Nothing to do.
			continue
		default:
			panic(fmt.Sprintf("unknown logical op %T", t))
		}

		//// Don't read previous values from the reader for operations that are
		//// not needed by any rangefeed registration.
		//if !filter.NeedPrevVal(roachpb.Span{Key: key}) {
		//	continue
		//}

		// Read the previous value from the prev Reader. Unlike the new value
		// (see handleLogicalOpLogRaftMuLocked), this one may be missing.
		//从raft中获取对应的befer value值，将其存储到Op中
		prevVal, _, err := engine.MVCCGet(
			ctx, prevReader, key, ts, engine.MVCCGetOptions{Tombstones: true, Inconsistent: true},
		)
		if err != nil {
			r.disconnectRangefeedWithErrRaftMuLocked(roachpb.NewErrorf(
				"error consuming %T for key %v @ ts %v: %v", op, key, ts, err,
			))
			return
		}
		if prevVal != nil {
			*prevValPtr = prevVal.RawBytes
		} else {
			*prevValPtr = nil
		}
	}
}




// handleLogicalOpLogRaftMuLocked passes the logical op log to the active
// rangefeed, if one is running. The method accepts a reader, which is used to
// look up the values associated with key-value writes in the log before handing
// them to the rangefeed processor. No-op if a rangefeed is not active. Requires
func (r *Replica) handleLogicalOpLogRaftMuLocked(
	ctx context.Context, ops *storagepb.LogicalOpLog, reader engine.Reader,
) {
	if r.raftMu.rangefeed == nil {
		return
	}
	if ops == nil {
		// Rangefeeds can't be turned on unless RangefeedEnabled is set to true,
		// after which point new Raft proposals will include logical op logs.
		// However, there's a race present where old Raft commands without a
		// logical op log might be passed to a rangefeed. Since the effect of
		// these commands was not included in the catch-up scan of current
		// registrations, we're forced to throw an error. The rangefeed clients
		// can reconnect at a later time, at which point all new Raft commands
		// should have logical op logs.
		r.disconnectRangefeedWithReasonRaftMuLocked(
			roachpb.RangeFeedRetryError_REASON_LOGICAL_OPS_MISSING,
		)
		return
	}
	if len(ops.Ops) == 0 {
		return
	}

	// When reading straight from the Raft log, some logical ops will not be
	// fully populated. Read from the Reader to populate all fields.
	for _, op := range ops.Ops {
		var key []byte
		var ts hlc.Timestamp
		var valPtr *[]byte
		switch t := op.GetValue().(type) {
		case *enginepb.MVCCWriteValueOp:
			key, ts, valPtr = t.Key, t.Timestamp, &t.Value
		case *enginepb.MVCCCommitIntentOp:
			key, ts, valPtr = t.Key, t.Timestamp, &t.Value
		case *enginepb.MVCCWriteIntentOp,
			*enginepb.MVCCUpdateIntentOp,
			*enginepb.MVCCAbortIntentOp,
			*enginepb.MVCCAbortTxnOp:
			// Nothing to do.
			continue
		default:
			panic(fmt.Sprintf("unknown logical op %T", t))
		}

		// Read the value directly from the Reader. This is performed in the
		// same raftMu critical section that the logical op's corresponding
		// WriteBatch is applied, so the value should exist.
		val, _, err := engine.MVCCGet(ctx, reader, key, ts, engine.MVCCGetOptions{Tombstones: true})
		if val == nil && err == nil {
			err = errors.New("value missing in reader")
		}
		if err != nil {
			r.disconnectRangefeedWithErrRaftMuLocked(roachpb.NewErrorf(
				"error consuming %T for key %v @ ts %v: %v", op, key, ts, err,
			))
			return
		}
		*valPtr = val.RawBytes
	}

	// Pass the ops to the rangefeed processor.
	//将opts发送到对应的rangfeed进行处理
	if !r.raftMu.rangefeed.ConsumeLogicalOps(ops.Ops...) {
		// Consumption failed and the rangefeed was stopped.
		r.resetRangefeedRaftMuLocked()
	}
}




// ConsumeLogicalOps informs the rangefeed processor of the set of logical
// operations. It returns false if consuming the operations hit a timeout, as
// specified by the EventChanTimeout configuration. If the method returns false,
// the processor will have been stopped, so calling Stop is not necessary. Safe
// to call on nil Processor.
func (p *Processor) ConsumeLogicalOps(ops ...enginepb.MVICLogicalOp) bool {
	if p == nil {
		return true
	}
	if len(ops) == 0 {
		return true
	}
	//将opts event 发送到对应的chan中进行处理
	return p.sendEvent(event{ops: ops}, p.EventChanTimeout)
}




// sendEvent informs the Processor of a new event. If a timeout is specified,
// the method will wait for no longer than that duration before giving up,
// shutting down the Processor, and returning false. 0 for no timeout.
func (p *Processor) sendEvent(e event, timeout time.Duration) bool {
	if timeout == 0 {
		select {
		case p.eventC <- e:
		case <-p.stoppedC:
			// Already stopped. Do nothing.
		}
	} else {
		select {
		case p.eventC <- e:
		case <-p.stoppedC:
			// Already stopped. Do nothing.
		default:
			select {
			case p.eventC <- e:
			case <-p.stoppedC:
				// Already stopped. Do nothing.
			case <-time.After(timeout):
				// Sending on the eventC channel would have blocked.
				// Instead, tear down the processor and return immediately.
				p.sendStop(newErrBufferCapacityExceeded())
				return false
			}
		}
	}
	return true
}






// Start launches a goroutine to process rangefeed events and send them to
// registrations.
//
// The provided iterator is used to initialize the rangefeed's resolved
// timestamp. It must obey the contract of an iterator used for an
// initResolvedTSScan. The Processor promises to clean up the iterator by
// calling its Close method when it is finished. If the iterator is nil then
// no initialization scan will be performed and the resolved timestamp will
// immediately be considered initialized.
//启动对应的RangeFeed Processor
func (p *Processor) Start(stopper *stop.Stopper, rtsIter engine.SimpleIterator) {
	ctx := p.AnnotateCtx(context.Background())
	stopper.RunWorker(ctx, func(ctx context.Context) {
		defer close(p.stoppedC)
		ctx, cancelOutputLoops := context.WithCancel(ctx)
		defer cancelOutputLoops()

		// Launch an async task to scan over the resolved timestamp iterator and
		// initialize the unresolvedIntentQueue. Ignore error if quiescing.
		if rtsIter != nil {
			initScan := newInitResolvedTSScan(p, rtsIter)
			err := stopper.RunAsyncTask(ctx, "rangefeed: init resolved ts", initScan.Run)
			if err != nil {
				initScan.Cancel()
			}
		} else {
			p.initResolvedTS(ctx)
		}

		// txnPushTicker periodically pushes the transaction record of all
		// unresolved intents that are above a certain age, helping to ensure
		// that the resolved timestamp continues to make progress.
		var txnPushTicker *time.Ticker
		var txnPushTickerC <-chan time.Time
		var txnPushAttemptC chan struct{}
		if p.PushTxnsInterval > 0 {
			txnPushTicker = time.NewTicker(p.PushTxnsInterval)
			txnPushTickerC = txnPushTicker.C
			defer txnPushTicker.Stop()
		}

		for {
			select {

			// Handle new registrations.
			case r := <-p.regC:
				if !p.Span.AsRawSpanWithNoLocals().Contains(r.span) {
					log.Fatalf(ctx, "registration %s not in Processor's key range %v", r, p.Span)
				}

				// Add the new registration to the registry.
				p.reg.Register(&r)

				// Immediately publish a checkpoint event to the registry. This will be
				// the first event published to this registration after its initial
				// catch-up scan completes.
				r.publish(p.newCheckpointEvent())

				// Run an output loop for the registry.
				runOutputLoop := func(ctx context.Context) {
					r.runOutputLoop(ctx)
					select {
					case p.unregC <- &r:
					case <-p.stoppedC:
					}
				}
				if err := stopper.RunAsyncTask(ctx, "rangefeed: output loop", runOutputLoop); err != nil {
					if r.catchupIter != nil {
						r.catchupIter.Close() // clean up
					}
					r.disconnect(roachpb.NewError(err))
					p.reg.Unregister(&r)
				}

			// Respond to unregistration requests; these come from registrations that
			// encounter an error during their output loop.
			case r := <-p.unregC:
				p.reg.Unregister(r)

			// Respond to answers about the processor goroutine state.
			case <-p.lenReqC:
				p.lenResC <- p.reg.Len()
			// Respond to answers about which operations can be filtered before
			// reaching the Processor.
			case <-p.filterReqC:
				p.filterResC <- p.reg.NewFilter()

			// Transform and route events.
			//消费对应的event并进行路由
			case e := <-p.eventC:
				p.consumeEvent(ctx, e)

			// Check whether any unresolved intents need a push.
			case <-txnPushTickerC:
				// Don't perform transaction push attempts until the resolved
				// timestamp has been initialized.
				if !p.rts.IsInit() {
					continue
				}

				now := p.Clock.Now()
				before := now.Add(-p.PushTxnsAge.Nanoseconds(), 0)
				oldTxns := p.rts.intentQ.Before(before)

				if len(oldTxns) > 0 {
					toPush := make([]enginepb.TxnMeta, len(oldTxns))
					for i, txn := range oldTxns {
						toPush[i] = txn.asTxnMeta()
					}

					// Set the ticker channel to nil so that it can't trigger a
					// second concurrent push. Create a push attempt response
					// channel that is closed when the push attempt completes.
					txnPushTickerC = nil
					txnPushAttemptC = make(chan struct{})

					// Launch an async transaction push attempt that pushes the
					// timestamp of all transactions beneath the push offset.
					// Ignore error if quiescing.
					pushTxns := newTxnPushAttempt(p, toPush, now, txnPushAttemptC)
					err := stopper.RunAsyncTask(ctx, "rangefeed: pushing old txns", pushTxns.Run)
					if err != nil {
						pushTxns.Cancel()
					}
				}

			// Update the resolved timestamp based on the push attempt.
			case <-txnPushAttemptC:
				// Reset the ticker channel so that it can trigger push attempts
				// again. Set the push attempt channel back to nil.
				txnPushTickerC = txnPushTicker.C
				txnPushAttemptC = nil

			// Close registrations and exit when signaled.
			case pErr := <-p.stopC:
				p.reg.DisconnectWithErr(all, pErr)
				return

			// Exit on stopper.
			case <-stopper.ShouldQuiesce():
				p.reg.Disconnect(all)
				return
			}
		}
	})
}


//消费opts event
func (p *Processor) consumeEvent(ctx context.Context, e event) {
	switch {
	case len(e.ops) > 0:
		p.consumeLogicalOps(ctx, e.ops)
	case e.ct != hlc.Timestamp{}:
		p.forwardClosedTS(ctx, e.ct)
	case e.initRTS:
		p.initResolvedTS(ctx)
	case e.syncC != nil:
		if e.testRegCatchupSpan.Valid() {
			if err := p.reg.waitForCaughtUp(e.testRegCatchupSpan); err != nil {
				log.Errorf(
					ctx,
					"error waiting for registries to catch up during test, results might be impacted: %s",
					err,
				)
			}
		}
		close(e.syncC)
	default:
		panic("missing event variant")
	}
}

func (p *Processor) consumeLogicalOps(ctx context.Context, ops []enginepb.MVICLogicalOp) {
	for _, op := range ops {
		// Publish RangeFeedValue updates, if necessary.
		switch t := op.GetValue().(type) {
		case *enginepb.MVCCWriteValueOp:
			// Publish the new value directly.
			p.publishValue(ctx, t.Key, t.Timestamp, t.Value, t.PrevValue)
		case *enginepb.MVCCWriteIntentOp:
			// No updates to publish.

		case *enginepb.MVCCUpdateIntentOp:
			// No updates to publish.

		case *enginepb.MVCCCommitIntentOp:
			// Publish the newly committed value.
			p.publishValue(ctx, t.Key, t.Timestamp, t.Value, t.PrevValue)

		case *enginepb.MVCCAbortIntentOp:
			// No updates to publish.

		case *enginepb.MVCCAbortTxnOp:
			// No updates to publish.

		default:
			panic(fmt.Sprintf("unknown logical op %T", t))
		}

		// Determine whether the operation caused the resolved timestamp to
		// move forward. If so, publish a RangeFeedCheckpoint notification.
		if p.rts.ConsumeLogicalOp(op) {
			p.publishCheckpoint(ctx)
		}
	}
}





// Register registers the stream over the specified span of keys.
//
// The registration will not observe any events that were consumed before this
// method was called. It is undefined whether the registration will observe
// events that are consumed concurrently with this call. The channel will be
// provided an error when the registration closes.
//
// The optionally provided "catch-up" iterator is used to read changes from the
// engine which occurred after the provided start timestamp.
//
// NOT safe to call on nil Processor.
func (p *Processor) Register(
	span roachpb.RSpan,
	startTS hlc.Timestamp,
	catchupIter engine.SimpleIterator,
	withDiff bool,
	stream Stream,
	errC chan<- *roachpb.Error,
) {
	// Synchronize the event channel so that this registration doesn't see any
	// events that were consumed before this registration was called. Instead,
	// it should see these events during its catch up scan.
	p.syncEventC()

	r := newRegistration(
		span.AsRawSpanWithNoLocals(), startTS, catchupIter, withDiff,
		p.Config.EventChanCap, p.Metrics, stream, errC,
	)
	select {
	case p.regC <- r:
	case <-p.stoppedC:
		if catchupIter != nil {
			catchupIter.Close() // clean up
		}
		// errC has a capacity of 1. If it is already full, we don't need to send
		// another error.
		select {
		case errC <- roachpb.NewErrorf("rangefeed processor closed"):
		default:
		}
	}
}



// RangeFeed registers a rangefeed over the specified span. It sends updates to
// the provided stream and returns with an optional error when the rangefeed is
// complete. The provided ConcurrentRequestLimiter is used to limit the number
// of rangefeeds using catchup iterators at the same time.
func (r *Replica) RangeFeed(
	args *roachpb.RangeFeedRequest, stream roachpb.Internal_RangeFeedServer,
) *roachpb.Error {
	if !RangefeedEnabled.Get(&r.store.cfg.Settings.SV) {
		return roachpb.NewErrorf("rangefeeds require the kv.rangefeed.enabled setting. See " +
			base.DocsURL(`change-data-capture.html#enable-rangefeeds-to-reduce-latency`))
	}
	ctx := r.AnnotateCtx(stream.Context())

	var rSpan roachpb.RSpan
	var err error
	rSpan.Key, err = keys.Addr(args.Span.Key)
	if err != nil {
		return roachpb.NewError(err)
	}
	rSpan.EndKey, err = keys.Addr(args.Span.EndKey)
	if err != nil {
		return roachpb.NewError(err)
	}

	if err := r.ensureClosedTimestampStarted(ctx); err != nil {
		return err
	}

	// If the RangeFeed is performing a catch-up scan then it will observe all
	// values above args.Timestamp. If the RangeFeed is requesting previous
	// values for every update then it will also need to look for the version
	// proceeding each value observed during the catch-up scan timestamp. This
	// means that the earliest value observed by the catch-up scan will be
	// args.Timestamp.Next and the earliest timestamp used to retrieve the
	// previous version of a value will be args.Timestamp, so this is the
	// timestamp we must check against the GCThreshold.
	checkTS := args.Timestamp
	if checkTS.IsEmpty() {
		// If no timestamp was provided then we're not going to run a catch-up
		// scan, so make sure the GCThreshold in requestCanProceed succeeds.
		checkTS = r.Clock().Now()
	}

	lockedStream := &lockedRangefeedStream{wrapped: stream}
	errC := make(chan *roachpb.Error, 1)

	// If we will be using a catch-up iterator, wait for the limiter here before
	// locking raftMu.
	usingCatchupIter := false
	var iterSemRelease func()
	if !args.Timestamp.IsEmpty() {
		usingCatchupIter = true
		lim := &r.store.limiters.ConcurrentRangefeedIters
		if err := lim.Begin(ctx); err != nil {
			return roachpb.NewError(err)
		}
		// Finish the iterator limit, but only if we exit before
		// creating the iterator itself.
		iterSemRelease = lim.Finish
		defer func() {
			if iterSemRelease != nil {
				iterSemRelease()
			}
		}()
	}

	// Lock the raftMu, then register the stream as a new rangefeed registration.
	// raftMu is held so that the catch-up iterator is captured in the same
	// critical-section as the registration is established. This ensures that
	// the registration doesn't miss any events.
	r.raftMu.Lock()
	if err := r.checkExecutionCanProceedForRangeFeed(rSpan, checkTS); err != nil {
		r.raftMu.Unlock()
		return roachpb.NewError(err)
	}

	// Ensure that the rangefeed processor is running.
	//注册对应RangeFeed Processor
	p := r.maybeInitRangefeedRaftMuLocked(ctx)

	// Register the stream with a catch-up iterator.
	var catchUpIter engine.SimpleIterator
	if usingCatchupIter {
		innerIter := r.Engine().NewIterator(engine.IterOptions{
			UpperBound: args.Span.EndKey,
			// RangeFeed originally intended to use the time-bound iterator
			// performance optimization. However, they've had correctness issues in
			// the past (#28358, #34819) and no-one has the time for the due-diligence
			// necessary to be confidant in their correctness going forward. Not using
			// them causes the total time spent in RangeFeed catchup on changefeed
			// over tpcc-1000 to go from 40s -> 4853s, which is quite large but still
			// workable. See #35122 for details.
			// MinTimestampHint: args.Timestamp,
		})
		catchUpIter = iteratorWithCloser{
			SimpleIterator: innerIter,
			close:          iterSemRelease,
		}
		// Responsibility for releasing the semaphore now passes to the iterator.
		iterSemRelease = nil
	}
	//注册一个registration,该registration为订阅特定范围内数据更新的示例，并通过outputLoop进行数据的流式传输
	p.Register(rSpan, args.Timestamp, catchUpIter, args.WithDiff, lockedStream, errC)
	r.raftMu.Unlock()

	// When this function returns, attempt to clean up the rangefeed.
	defer func() {
		r.raftMu.Lock()
		r.maybeDestroyRangefeedRaftMuLocked(p)
		r.raftMu.Unlock()
	}()

	// Block on the registration's error channel. Note that the registration
	// observes stream.Context().Done.
	return <-errC
}

// outputLoop is the operational loop for a single registration. The behavior
// is as thus:
//
// 1. If a catch-up scan is indicated, run one before beginning the proper
// output loop.
// 2. After catch-up is complete, begin reading from the registration buffer
// channel and writing to the output stream until the buffer is empty *and*
// the overflow flag has been set.
//
// The loop exits with any error encountered, if the provided context is
// canceled, or when the buffer has overflowed and all pre-overflow entries
// have been emitted.
func (r *registration) outputLoop(ctx context.Context) error {
	// If the registration has a catch-up scan,
	if r.catchupIter != nil {
		if err := r.runCatchupScan(); err != nil {
			err = errors.Wrap(err, "catch-up scan failed")
			log.Error(ctx, err)
			return err
		}
	}

	// Normal buffered output loop.
	for {
		overflowed := false
		r.mu.Lock()
		if len(r.buf) == 0 {
			overflowed = r.mu.overflowed
			r.mu.caughtUp = true
		}
		r.mu.Unlock()
		if overflowed {
			return newErrBufferCapacityExceeded().GoError()
		}

		select {
		case nextEvent := <-r.buf:
			if err := r.stream.Send(nextEvent); err != nil {
				return err
			}
		case <-ctx.Done():
			return ctx.Err()
		case <-r.stream.Context().Done():
			return r.stream.Context().Err()
		}
	}
}




// RangeFeed registers a rangefeed over the specified span. It sends updates to
// the provided stream and returns with an optional error when the rangefeed is
// complete.
//store 上的RangeFeed接口，下发到指定Range上开启RangeFeed
func (s *Store) RangeFeed(
	args *roachpb.RangeFeedRequest, stream roachpb.Internal_RangeFeedServer,
) *roachpb.Error {
	if err := verifyKeys(args.Span.Key, args.Span.EndKey, true); err != nil {
		return roachpb.NewError(err)
	}

	// Get range and add command to the range for execution.
	repl, err := s.GetReplica(args.RangeID)
	if err != nil {
		return roachpb.NewError(err)
	}
	if !repl.IsInitialized() {
		repl.mu.RLock()
		replicaID := repl.mu.replicaID
		repl.mu.RUnlock()

		// If we have an uninitialized copy of the range, then we are
		// probably a valid member of the range, we're just in the
		// process of getting our snapshot. If we returned
		// RangeNotFoundError, the client would invalidate its cache,
		// but we can be smarter: the replica that caused our
		// uninitialized replica to be created is most likely the
		// leader.
		return roachpb.NewError(&roachpb.NotLeaseHolderError{
			RangeID:     args.RangeID,
			LeaseHolder: repl.creatingReplica,
			// The replica doesn't have a range descriptor yet, so we have to build
			// a ReplicaDescriptor manually.
			Replica: roachpb.ReplicaDescriptor{
				NodeID:    repl.store.nodeDesc.NodeID,
				StoreID:   repl.store.StoreID(),
				ReplicaID: replicaID,
			},
		})
	}
	return repl.RangeFeed(args, stream)
}



// RangeFeed registers a rangefeed over the specified span. It sends updates to
// the provided stream and returns with an optional error when the rangefeed is
// complete.
//Stores级别的RangeFeed接口，将RangeFeed请求下发到对应的Store上
func (ls *Stores) RangeFeed(
	args *roachpb.RangeFeedRequest, stream roachpb.Internal_RangeFeedServer,
) *roachpb.Error {
	ctx := stream.Context()
	if args.RangeID == 0 {
		log.Fatal(ctx, "rangefeed request missing range ID")
	} else if args.Replica.StoreID == 0 {
		log.Fatal(ctx, "rangefeed request missing store ID")
	}

	store, err := ls.GetStore(args.Replica.StoreID)
	if err != nil {
		return roachpb.NewError(err)
	}

	return store.RangeFeed(args, stream)
}


// RangeFeed implements the roachpb.InternalServer interface.
//Node对应的RangeFeed接口，将请求下发到Stores
func (n *Node) RangeFeed(
	args *roachpb.RangeFeedRequest, stream roachpb.Internal_RangeFeedServer,
) error {
	pErr := n.stores.RangeFeed(args, stream)
	if pErr != nil {
		var event roachpb.RangeFeedEvent
		event.SetValue(&roachpb.RangeFeedError{
			Error: *pErr,
		})
		return stream.Send(&event)
	}
	return nil
}

//grpc server RangeFeed接口，通过InternalClient调用
func (a internalClientAdapter) RangeFeed(
	ctx context.Context, args *roachpb.RangeFeedRequest, _ ...grpc.CallOption,
) (roachpb.Internal_RangeFeedClient, error) {
	ctx, cancel := context.WithCancel(ctx)
	rfAdapter := rangeFeedClientAdapter{
		ctx:    ctx,
		eventC: make(chan *roachpb.RangeFeedEvent, 128),
		errC:   make(chan error, 1),
	}

	go func() {
		defer cancel()
		err := a.InternalServer.RangeFeed(args, rfAdapter)
		if err == nil {
			err = io.EOF
		}
		rfAdapter.errC <- err
	}()

	return rfAdapter, nil
}



// singleRangeFeed gathers and rearranges the replicas, and makes a RangeFeed
// RPC call. Results will be send on the provided channel. Returns the timestamp
// of the maximum rangefeed checkpoint seen, which can be used to re-establish
// the rangefeed with a larger starting timestamp, reflecting the fact that all
// values up to the last checkpoint have already been observed. Returns the
// request's timestamp if not checkpoints are seen.
//分发层对应的RangeFeed接口，生成对应的GRPC请求。将RangeFeed请求下发到对应节点的对应store上的对应replica
func (ds *DistSender) singleRangeFeed(
	ctx context.Context,
	span roachpb.Span,
	ts hlc.Timestamp,
	withDiff bool,
	desc *roachpb.RangeDescriptor,
	eventCh chan<- *roachpb.RangeFeedEvent,
) (hlc.Timestamp, *roachpb.Error) {
	args := roachpb.RangeFeedRequest{
		Span: span,
		Header: roachpb.Header{
			Timestamp: ts,
			RangeID:   desc.RangeID,
		},
		WithDiff: withDiff,
	}

	var latencyFn LatencyFunc
	if ds.rpcContext != nil {
		latencyFn = ds.rpcContext.RemoteClocks.Latency
	}
	replicas := NewReplicaSlice(ds.gossip, desc)
	//对于replicas进行排序
	replicas.OptimizeReplicaOrder(ds.getNodeDescriptor(), latencyFn)

	transport, err := ds.transportFactory(SendOptions{}, ds.nodeDialer, replicas)
	if err != nil {
		return args.Timestamp, roachpb.NewError(err)
	}

	for {
		if transport.IsExhausted() {
			return args.Timestamp, roachpb.NewError(roachpb.NewSendError(
				fmt.Sprintf("sending to all %d replicas failed", len(replicas)),
			))
		}

		args.Replica = transport.NextReplica()
		clientCtx, client, err := transport.NextInternalClient(ctx)
		if err != nil {
			log.VErrEventf(ctx, 2, "RPC error: %s", err)
			continue
		}
		//调用对应的GRPC接口发送RangeFeed请求
		stream, err := client.RangeFeed(clientCtx, &args)
		if err != nil {
			log.VErrEventf(ctx, 2, "RPC error: %s", err)
			continue
		}
		for {
			event, err := stream.Recv()
			if err == io.EOF {
				return args.Timestamp, nil
			}
			if err != nil {
				return args.Timestamp, roachpb.NewError(err)
			}
			switch t := event.GetValue().(type) {
			case *roachpb.RangeFeedCheckpoint:
				if t.Span.Contains(args.Span) {
					args.Timestamp.Forward(t.ResolvedTS)
				}
			case *roachpb.RangeFeedError:
				log.VErrEventf(ctx, 2, "RangeFeedError: %s", t.Error.GoError())
				return args.Timestamp, &t.Error
			}
			select {
			case eventCh <- event:
			case <-ctx.Done():
				return args.Timestamp, roachpb.NewError(ctx.Err())
			}
		}
	}
}





// partialRangeFeed establishes a RangeFeed to the range specified by desc. It
// manages lifecycle events of the range in order to maintain the RangeFeed
// connection; this may involve instructing higher-level functions to retry
// this rangefeed, or subdividing the range further in the event of a split.
func (ds *DistSender) partialRangeFeed(
	ctx context.Context,
	rangeInfo *singleRangeInfo,
	withDiff bool,
	rangeCh chan<- singleRangeInfo,
	eventCh chan<- *roachpb.RangeFeedEvent,
) error {
	// Bound the partial rangefeed to the partial span.
	span := rangeInfo.rs.AsRawSpanWithNoLocals()
	ts := rangeInfo.ts

	// Start a retry loop for sending the batch to the range.
	for r := retry.StartWithCtx(ctx, ds.rpcRetryOptions); r.Next(); {
		// If we've cleared the descriptor on a send failure, re-lookup.
		if rangeInfo.desc == nil {
			var err error
			rangeInfo.desc, rangeInfo.token, err = ds.getDescriptor(ctx, rangeInfo.rs.Key, nil, false)
			if err != nil {
				log.VErrEventf(ctx, 1, "range descriptor re-lookup failed: %s", err)
				continue
			}
		}

		// Establish a RangeFeed for a single Range.
		maxTS, pErr := ds.singleRangeFeed(ctx, span, ts, withDiff, rangeInfo.desc, eventCh)

		// Forward the timestamp in case we end up sending it again.
		ts.Forward(maxTS)

		if pErr != nil {
			if log.V(1) {
				log.Infof(ctx, "RangeFeed %s disconnected with last checkpoint %s ago: %v",
					span, timeutil.Since(ts.GoTime()), pErr)
			}
			switch t := pErr.GetDetail().(type) {
			case *roachpb.SendError, *roachpb.RangeNotFoundError:
				// Evict the decriptor from the cache and reload on next attempt.
				if err := rangeInfo.token.Evict(ctx); err != nil {
					return err
				}
				rangeInfo.desc = nil
				continue
			case *roachpb.RangeKeyMismatchError:
				// Evict the decriptor from the cache.
				if err := rangeInfo.token.Evict(ctx); err != nil {
					return err
				}
				//出现range key范围不匹配时对RangeFeed重新进行划分并发送到各个Range上，
				return ds.divideAndSendRangeFeedToRanges(ctx, rangeInfo.rs, ts, rangeCh)
			case *roachpb.RangeFeedRetryError:
				switch t.Reason {
				case roachpb.RangeFeedRetryError_REASON_REPLICA_REMOVED,
					roachpb.RangeFeedRetryError_REASON_RAFT_SNAPSHOT,
					roachpb.RangeFeedRetryError_REASON_LOGICAL_OPS_MISSING,
					roachpb.RangeFeedRetryError_REASON_SLOW_CONSUMER:
					// Try again with same descriptor. These are transient
					// errors that should not show up again.
					continue
				case roachpb.RangeFeedRetryError_REASON_RANGE_SPLIT,
					roachpb.RangeFeedRetryError_REASON_RANGE_MERGED:
					// Evict the decriptor from the cache.
					if err := rangeInfo.token.Evict(ctx); err != nil {
						return err
					}
					//出现range split或者merged 对RangeFeed重新进行划分并发送到各个Range上，
					return ds.divideAndSendRangeFeedToRanges(ctx, rangeInfo.rs, ts, rangeCh)
				default:
					log.Fatalf(ctx, "unexpected RangeFeedRetryError reason %v", t.Reason)
				}
			default:
				return t
			}
		}
	}
	return nil
}

//DistSender 对应的RangeFeed接口
// RangeFeed divides a RangeFeed request on range boundaries and establishes a
// RangeFeed to each of the individual ranges. It streams back results on the
// provided channel.
func (ds *DistSender) RangeFeed(
	ctx context.Context,
	args *roachpb.RangeFeedRequest,
	withDiff bool,
	eventCh chan<- *roachpb.RangeFeedEvent,
) error {
	ctx = ds.AnnotateCtx(ctx)
	ctx, sp := tracing.EnsureChildSpan(ctx, ds.AmbientContext.Tracer, "dist sender")
	defer sp.Finish()

	startRKey, err := keys.Addr(args.Span.Key)
	if err != nil {
		return err
	}
	endRKey, err := keys.Addr(args.Span.EndKey)
	if err != nil {
		return err
	}
	rs := roachpb.RSpan{Key: startRKey, EndKey: endRKey}

	g := ctxgroup.WithContext(ctx)
	// Goroutine that processes subdivided ranges and creates a rangefeed for
	// each.
	rangeCh := make(chan singleRangeInfo, 16)
	g.GoCtx(func(ctx context.Context) error {
		for {
			select {
			case sri := <-rangeCh:
				// Spawn a child goroutine to process this feed.
				g.GoCtx(func(ctx context.Context) error {
					return ds.partialRangeFeed(ctx, &sri, withDiff, rangeCh, eventCh)
				})
			case <-ctx.Done():
				return ctx.Err()
			}
		}
	})

	// Kick off the initial set of ranges.
	g.GoCtx(func(ctx context.Context) error {
		return ds.divideAndSendRangeFeedToRanges(ctx, rs, args.Timestamp, rangeCh)
	})

	return g.Wait()
}

//主要负责rangfeed接口调用以及rangfeed推出数据的处理，回填处理等
func (p *poller) rangefeedImplIter(ctx context.Context, i int) error {
	// Determine whether to request the previous value of each update from
	// RangeFeed based on whether the `diff` option is specified.
	_, withDiff := p.details.Opts[optDiff]

	p.mu.Lock()
	lastHighwater := p.mu.highWater
	p.mu.Unlock()
	if err := p.tableHist.WaitForTS(ctx, lastHighwater); err != nil {
		return err
	}

	spans, err := getSpansToProcess(ctx, p.db, p.spans)
	if err != nil {
		return err
	}

	// Perform a full scan if necessary - either an initial scan or a backfill
	// Full scans are still performed using an Export operation.
	initialScan := i == 0
	backfillWithDiff := !initialScan && withDiff
	var scanTime hlc.Timestamp
	p.mu.Lock()
	if len(p.mu.scanBoundaries) > 0 && p.mu.scanBoundaries[0].Equal(p.mu.highWater) {
		// Perform a full scan of the latest value of all keys as of the
		// boundary timestamp and consume the boundary.
		scanTime = p.mu.scanBoundaries[0]
		p.mu.scanBoundaries = p.mu.scanBoundaries[1:]
	}
	p.mu.Unlock()
	if scanTime != (hlc.Timestamp{}) {
		// TODO(dan): Now that we no longer have the poller, we should stop using
		// ExportRequest and start using normal Scans.
		if err := p.exportSpansParallel(ctx, spans, scanTime, backfillWithDiff); err != nil {
			return err
		}
	}
	// Start rangefeeds, exit polling if we hit a resolved timestamp beyond
	// the next scan boundary.
	// TODO(nvanbenschoten): This is horrible.
	sender := p.db.NonTransactionalSender()
	ds := sender.(*client.CrossRangeTxnWrapperSender).Wrapped().(*kv.DistSender)
	g := ctxgroup.WithContext(ctx)
	eventC := make(chan *roachpb.RangeFeedEvent, 128)

	// To avoid blocking raft, RangeFeed puts all entries in a server side
	// buffer. But to keep things simple, it's a small fixed-sized buffer. This
	// means we need to ingest everything we get back as quickly as possible, so
	// we throw it in a buffer here to pick up the slack between RangeFeed and
	// the sink.
	//
	// TODO(dan): Right now, there are two buffers in the changefeed flow when
	// using RangeFeeds, one here and the usual one between the poller and the
	// rest of the changefeed (he latter of which is implemented with an
	// unbuffered channel, and so doesn't actually buffer). Ideally, we'd have
	// one, but the structure of the poller code right now makes this hard.
	// Specifically, when a schema change happens, we need a barrier where we
	// flush out every change before the schema change timestamp before we start
	// emitting any changes from after the schema change. The poller's
	// `tableHist` is responsible for detecting and enforcing these (they queue
	// up in `p.scanBoundaries`), but the after-poller buffer doesn't have
	// access to any of this state. A cleanup is in order.
	memBuf := makeMemBuffer(p.mm.MakeBoundAccount(), p.metrics)
	defer memBuf.Close(ctx)

	// Maintain a local spanfrontier to tell when all the component rangefeeds
	// being watched have reached the Scan boundary.
	// TODO(mrtracy): The alternative to this would be to maintain two
	// goroutines for each span (the current arrangement is one goroutine per
	// span and one multiplexing goroutine that outputs to the buffer). This
	// alternative would allow us to stop the individual rangefeeds earlier and
	// avoid the need for a span frontier, but would also introduce a different
	// contention pattern and use additional goroutines. it's not clear which
	// solution is best without targeted performance testing, so we're choosing
	// the faster-to-implement solution for now.
	frontier := makeSpanFrontier(spans...)

	rangeFeedStartTS := lastHighwater
	for _, span := range p.spans {
		req := &roachpb.RangeFeedRequest{
			Header: roachpb.Header{
				Timestamp: lastHighwater,
			},
			Span: span,
		}
		frontier.Forward(span, rangeFeedStartTS)
		g.GoCtx(func(ctx context.Context) error {
		    //调用分发层对应的RangeFeed接口
			return ds.RangeFeed(ctx, req, withDiff, eventC)
		})
	}
	g.GoCtx(func(ctx context.Context) error {
		for {
			select {
			case e := <-eventC:
				switch t := e.GetValue().(type) {
				case *roachpb.RangeFeedValue:
					kv := roachpb.KeyValue{Key: t.Key, Value: t.Value}
					var prevVal roachpb.Value
					if withDiff {
						prevVal = t.PrevValue
					}
					//将变更信息填充到memBuf中以供接下来进行消费
					if err := memBuf.AddKV(ctx, kv, prevVal, hlc.Timestamp{}); err != nil {
						return err
					}
				case *roachpb.RangeFeedCheckpoint:
					if !t.ResolvedTS.IsEmpty() && t.ResolvedTS.Less(rangeFeedStartTS) {
						// RangeFeed happily forwards any closed timestamps it receives as
						// soon as there are no outstanding intents under them.
						// Changefeeds don't care about these at all, so throw them out.
						continue
					}
					if err := memBuf.AddResolved(ctx, t.Span, t.ResolvedTS); err != nil {
						return err
					}
				default:
					log.Fatalf(ctx, "unexpected RangeFeedEvent variant %v", t)
				}
			case <-ctx.Done():
				return ctx.Err()
			}
		}
	})
	//进行变更信息的消费
	g.GoCtx(func(ctx context.Context) error {
		for {
			e, err := memBuf.Get(ctx)
			if err != nil {
				return err
			}
			if e.kv.Key != nil {
				if err := p.tableHist.WaitForTS(ctx, e.kv.Value.Timestamp); err != nil {
					return err
				}
				pastBoundary := false
				p.mu.Lock()
				if len(p.mu.scanBoundaries) > 0 && p.mu.scanBoundaries[0].Less(e.kv.Value.Timestamp) {
					// Ignore feed results beyond the next boundary; they will be retrieved when
					// the feeds are restarted after the scan.
					pastBoundary = true
				}
				p.mu.Unlock()
				if pastBoundary {
					continue
				}
				if err := p.buf.AddKV(ctx, e.kv, e.prevVal, e.schemaTimestamp, hlc.Timestamp{}); err != nil {
					return err
				}
			} else if e.resolved != nil {
				resolvedTS := e.resolved.Timestamp
				boundaryBreak := false
				// Make sure scan boundaries less than or equal to `resolvedTS` were
				// added to the `scanBoundaries` list before proceeding.
				if err := p.tableHist.WaitForTS(ctx, resolvedTS); err != nil {
					return err
				}
				p.mu.Lock()
				if len(p.mu.scanBoundaries) > 0 && !resolvedTS.Less(p.mu.scanBoundaries[0]) {
					boundaryBreak = true
					resolvedTS = p.mu.scanBoundaries[0]
				}
				p.mu.Unlock()
				if boundaryBreak {
					// A boundary here means we're about to do a full scan (backfill)
					// at this timestamp, so at the changefeed level the boundary
					// itself is not resolved. Skip emitting this resolved timestamp
					// because we want to trigger the scan first before resolving its
					// scan boundary timestamp.
					resolvedTS = resolvedTS.Prev()
					frontier.Forward(e.resolved.Span, resolvedTS)
					if frontier.Frontier() == resolvedTS {
						// All component rangefeeds are now at the boundary.
						// Break out of the ctxgroup by returning a sentinel error.
						return errBoundaryReached
					}
				} else {
					if err := p.buf.AddResolved(ctx, e.resolved.Span, resolvedTS); err != nil {
						return err

					}
				}
			}
		}
	})
	// TODO(mrtracy): We are currently tearing down the entire rangefeed set in
	// order to perform a scan; however, given that we have an intermediate
	// buffer, its seems that we could do this without having to destroy and
	// recreate the rangefeeds.
	if err := g.Wait(); err != nil && err != errBoundaryReached {
		return err
	}
	p.mu.Lock()
	p.mu.highWater = p.mu.scanBoundaries[0]
	p.mu.Unlock()
	return nil
}
```




# CDC DDL捕获设计文档



## Online Schema Change分析
关系型数据库中数据以表为单位存储，读写操作依赖表的模式(Schema)和数据(Data)。DDL 通常需要同时修改模式和数据，为了避免并发问题，早期的数据库实现会在 DDL 过程中禁止目标表上的读写操作（简称锁表）。锁表保证了对模式和数据的修改是原子操作。但某些 DDL 操作涉及复制数据（比如索引构建）执行时间可能在几分钟甚至数小时。OLTP 数据库中长时间锁表会对业务产生不可预知的影响，生产环境中不可接受，因此“能够与读写操作并行执行”是 OLTP 用户对 DDL 的核心需求，也是 Online Schema Change 要解决的问题。

### 分布式数据库
分布式数据库通常是一个集群，出于性能考虑，每个节点需要缓存一份 Schema。如果继续采用单机数据库的 DDL 流程，则需要通过分布式锁来保证加载新版本 Schema 过程中没有读写操作进行，代价极高，并且当集群内节点不能够互相感知时将变为无法完成的任务。
![distribute.jpg](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/distribute.jpg)

讨论解决方案之前，以 CREATE INDEX 为例，看看集群节点使用不同版本 Schema 执行读写操作，带来的具体问题。
![progrems.jpg](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/progrems.jpg)
上图展示的是一个存储计算分离架构的分布式数据库，集群由 CN(计算节点) 和 DN(存储节点) 构成，每个 CN 中缓存一份 Schema。由于 CN0 和 CN1 异步加载 Schema，添加索引过程中可能存在一个时刻，CN0 认为有索引而 CN1 认为没有，此时产生两种异常

索引上有多余数据(Orphan Data Anomaly): CN0 执行了 INSERT，在表和索引上插入数据，随后 CN1 执行 DELETE。由于 CN1 认为没有索引，仅删除了表上的数据
索引上缺少数据(Integrity Anomaly): CN1 执行 INSERT，由于 CN1 认为没有索引，仅在表上插入数据，没有写相关的增量日志，导致索引创建完成后缺少这次 INSERT 的数据
可以看到，如果同一时刻存在两个 Schema 版本的情况无法避免，继续沿用单机数据库一步完成 Schema 版本切换的方案，会导致数据问题。那么如果“一步”切换不可行，“多步”能否解决问题？VLDB 2013 上 Google 工程师给出了一种新的 Schema Change 流程，通过增加两个中间状态来解决这个问题[7]。

### F1 Online Schema Change
Google F1 的方案引入了两个中间状态，delete_only 状态的对象上仅执行删除操作，write_only 状态的对象上支持写入，但不允许读取。依然以 CREATE INDEX 为例

解决 Orphan Data Anomaly：CN0 认为索引处于 delete_only 状态，仅在表上插入数据，CN1 认为没有索引，仅在表上删除数据。最终索引和表上都没有 id = 0 的数据
解决 Integrity Anomaly：CN1 认为索引处于 delete_only 状态，仅在表上插入数据，没有写相关的增量日志，但由于还有节点没有更新到 V2 版本，数据回填没有开始。当所有节点都更新 V2 版本后，数据回填操作会在索引中填入这一条数据
![f1-online-schema-change.jpg](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/f1-online-schema-change.jpg)
以上两个具体场景为例，说明了方案的有效性。

#### Google F1 的方案，包含两个关键点：

* 增加两个中间状态(delete_only，write_only)，允许集群中的事务同时使用至多两个最近的元数据版本

* 增加租约(lease)的概念，保证在一个租约周期内，没有拿到最新版本 Schema 的节点，无法提交事务

第一个关键点，将保证 Schema Change 正确性的条件，从“只能有一个版本”，降低为“最多可以有两个版本”。第二个关键点，给出了 F1 系统“保证最多两个版本”的实现思路。
简单来说，Google F1 的方案，成功将问题“在分布式数据库系统上实现 Online Schema Change”转化为“设计一种保证系统中最多有两个 Schema 版本的协议”，并且给出了一种基于租约的协议实现。
协议内容可以概括为，以租约周期作为时间单位，协调了三个操作的节奏
- Schema 刷新的间隔：每个节点需要在租约过期前，获取一次最新版本 Schema，如果无法获取，则主动退出，由托管服务重新拉起
- DDL 的最小时长：每次更新 Schema 版本后，需要等待一个租约周期，保证所有节点都读到最新版本的元数据
- 事务的最大时长：执行时间超过一个租约周期的事务将被回滚，确保事务仅使用了一个 Schema 版本
原始版本的协议十分简洁，易于描述和验证，但由于将 DDL 执行的最小时长和事务执行的最大时长绑定在一起，使用体验上与单机数据库有区别。对此，业界也给出了多种改进方案，比如：

#### CockroachDB 重新设计了 schema lease[8] ，在两方面做出改进：
- 降低 DDL 执行的最小时长：通过在事务开始时获取一个包含版本信息的租约，事务结束时释放，使得更新 Schema 版本后能够立即确认旧版本是否还在被使用，仅在有长事务或者节点异常（比如网络断开）时才需要等满一个租约周期。由于通过在存储中插入记录来获取租约，会增加事务的执行耗时。
- 只在 DDL 执行过程中限制事务的最大时长：具体做法是，使用 Schema 版本变更开始的时间作为边界，产生一个 [Tv,Tv+2) 的时间窗口。起始时间在窗口内，结束时间在窗口外的事务将被回滚。如果有 DDL 正在执行，则窗口最大为两个租约周期。如果没有 DDL 执行，则不存在 v+2 版本，可以认为是一个无限大的窗口[Tv,+∞)

## CDC DDL捕获原理
总的来说，CDC DDL捕获设计与DML捕获的基本原理类似。
![cdc-seq.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/cdc-seq.png)

### CDC DDL数据来源
首先，DDL捕获开启一个特殊的changeAggregator，该changeAggregator主要负责监控一个特殊的表-***system.descriptor***,如上述对于Online Schema Change的分析，一次Schema Change包含多不操作，也即对应的 ***system.descriptor*** 具有多次变更记录。而对于CDC DDL捕获功能来说，目前仅需要实际设置为 ***public***  状态的变更记录。因此对 ***system.descriptor*** 触发出的数据进行相关处理操作。
### CDC DDL DML顺序问题解决
![cdc ddl.png](https://cdn.jsdelivr.net/gh/shimengfei/cdn/img/cdc ddl.png)


#### DDL

* DDL阻塞条件
resolvedTS < lastDMLTransactionCommitTime

* DDL放出条件
resolvedTS >= lastDMLTransactionCommitTime
 **变量解释：**

    1. resolvedTS
    变量含义：该表对应的已解析时间戳
    更新位置：该表DML已经不再发送将该表对应的span标志为resolvedSpan
    1. lastDMLTransactionCommitTime
    变量含义：DML事务提交时间
    更新位置：DML成功发送后，更新DML事务提交时间

#### DML

* DML需写入临时文件条件

    lastDDLSendTime < changefeedDesc.ModificationTime

* DML直接发送条件

    lastDDLSendTime >= changefeedDesc.ModificationTime
    **变量解释：**

    1. lastDDLSendTime
    变量含义：DDL上次发送对应的表的ModifyTime
    更新位置：DDL成功发送后，更新lastDDLSendTime为ModifyTime
    1.  changefeedDesc.ModificationTime
    变量含义：该次DML对应的ModificationTime

### DDL&DML Model
   代表俩个模式，分别负责DDL发送与DML发送。

1. 处于DDL Model:每次发送DDL后，读取临时文件，如果临时文件中的DML满足DML直接发送条件，则切换为DML Model。
1. 处于DML Model:DML更新resolvedTS后，如果满足DDL直接发送条件，则切换为DDL Model。
# CDC 具有外键约束表数据顺序捕获分析


## 具有外键约束表的数据变更特点
* 数据变更为同一事务
* 主从表数据需要顺序发送


## 具有外键约束表的数据变更捕获方案
总的来说就是进行排序操作，按照事务的时间戳ts进行重组。然后主表先入从表后入

主要问题为：
* 基于目前的CDC体系在数据库内部进行排序以及事务重组操作将会特别消耗数据库资源，再未进行排序时，开启CDC大概会消耗5%~15%左右的数据库性能，如果这些操作在内部进行处理时，会导致更大的性能消耗


### 注意
因为ZNbase 本身使用的是混合逻辑时钟，所以两个不重叠事务的时间戳可能会相同，但是混合逻辑时钟具有纳秒级的精度。这个概率较小。

