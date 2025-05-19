---
layout: page
title: Zeze漫谈
parent: intro
nav_order: 1
---

* TOC
{:toc}



# Zeze漫谈

---

## 简介

Zeze是一个嵌入开发语言的分布式事务框架。

## 思想变化和规则

- 简单
- 直接
- 暴力
- 耦合，综合上面的形成了这一点，或解开或合并，没有纯粹单一的规则。
- 刚好原则
  - 考虑未来，但不多做东西，需要的时候再来添加。
- 分层思路、二的魔法
  - 考虑某个局部系统时，一般只考虑二级。从整个系统，最终还是多级的。

## 庞大的记录(Record)

Record最早只是Cache中的一个数据，后来所有相关功能需要的时候，都会在Record里面加上自己的状态。现在已经变得很庞大。

### 1. class Record

#### a) 乐观锁和缓存

下面变量定义按源码顺序排列，没有分类。

```
ReentrantLock fairLock;                       // 记录锁。
boolean dirty;                                // 记录是否脏的标志。
volatile Bean strongDirtyValue;               // 脏记录用这个保存数据Bean的引用，防止脏记录被软引用回收掉。
volatile long timestamp;                      // 时戳，乐观锁核心状态。
volatile SoftReference<Bean> softValue;       // 记录软引用，满足一定条件可以被回收，被回收的数据Bean会在需要的时候重新重本地硬盘中装载进来。
volatile RelativeRecordSet relativeRecordSet; // 记录所属的关联记录集合。记录总是属于一个关联记录集合。
volatile int state;                           // 记录状态，也就是是否获得锁的标志。
```

SoftReference 二级缓存

记录缓存最早的版本都保存在内存中的。通过每个表的缓存容量配置（CacheCapacity）控制内存的使用。
由于表的记录大小不一，使用频率不一，通过配置控制内存使用是一个比较繁重的工作。所以，引入基于
软引用的二级缓存，内存中用SoftReference<Bean>，同时在本地硬盘保存一份。当软引用失效时从本地硬盘装载。
引入软引用以后，JVM会最大化利用内存，把部分记录回收掉。这样每个记录在内存中只记录一个引用，
没有数据，缓存容量就可以设置的比较大，内存也不容易受到记录大小的影响。带来的好处是，I) 使得本地
缓存更大，有利于提高缓存命中率；II) 简化配置，即统一把所有表的容量都设置成一样的，并且比较大的数值。

乐观锁和关联记录集合相关的后面再说。

#### b) 新鲜的优化

```
boolean fresh; 
long acquireTime;
// 用来表示记录是否新鲜的。刚刚获得控制权的记录，由于乐观锁，有可能没有真正成功执行完事务又会被其他服务器抢走，
// 造成资源浪费，甚至一直互相抢夺造成事务失败，虽然可能性比较低。

final boolean isFresh() {
	// 这个标志在申请锁时会被发送给Global，并影响Global对锁的分配。
	return fresh;
}

final boolean isFreshAcquire() {
	// 新鲜的，并且获得时间小于1秒，此时别的服务器的申请会被拒绝。
	// fresh 被降级时会设置成false；
	// acquireTime 是一个保护措施，防止获得锁后最终没有实际使用的情况，超过一秒也会被认为不再新鲜；
	return fresh && System.currentTimeMillis() - acquireTime < 1000;
}

final void setNotFresh() {
	// 被降级，锁被分配给别的服务器了。不再新鲜。
	fresh = false;
}

final void setFreshAcquire() {
	// 申请成功装载记录时设置。
	acquireTime = System.currentTimeMillis();
	fresh = true;
}
```

#### c) Checkpoint的优化
```
	Database.Transaction databaseTransactionTmp;
	Database.Transaction databaseTransactionOldTmp;
```

- Database.Transaction 
    是Zeze的后端数据库，如MySql的事务。
	Zeze支持多个后端数据库，并把他们看成一个整体来使用。
	Checkpoint时，会创建每个后端数据库的事务用来提交数据。
	Checkpoint是按记录操作的，把后端数据库的事务保存到记录上是一种优化，
	这样保存记录数据时，不用一个个去查找后端数据库的事务了。

- databaseTransactionOldTmp
	Zeze支持的一个策略：可以把一个很大的后端数据库备份放到一边变成"只读"，
	Zeze会在执行中，把数据从备份库中根据实际使用装载到当前的工作数据库中。
	对于某些游戏不断合服，造成后端数据库很大，但是活跃数据又不大时，这个
	策略可以大大的减小工作数据库，有利于性能和减少全备份时间。
	使用这个策略，主要是修改配置，由于这个功能比较奇怪，这里不详细说明配置了。
	实际上一下子也说不出来，需要去看代码。"只读"，当记录从工作记录中删除时，
	需要删除备份库，否则下一次查询又装载一次，逻辑就不对了。

### 2. class Record1<K, V> extends Record

这个类是上面Record的子类，带了类型，以及系列化相关的方法。


- a) 基本
	TableX<K, V> table;       // 记录所属的表。
	final K key;              // 记录的Key。

- b) 系列化快照
	ByteBuffer snapshotKey;   // 记录的Key系列化后的临时引用。
	ByteBuffer snapshotValue; // 记录的Value系列化后的临时引用。

- c) 安全清除Dirty
	long savedTimestampForCheckpointPeriod; // 用来保存Checkpoint关键点时的时戳。
	//      记录的修改访问操作是和Checkpoint流程部分流程并发的，存在下面的情况。
	//      当记录建立快照后，又会被修改时，Checkpoint不能把记录设置成不脏。
	//      Checkpoint仅在保存的时戳和快照时的时戳一样时才修改记录的脏标志。

  - d) 删除不存在的记录优化
    boolean existInBackDatabase;                    // 记录是否在后端数据库中存在。
    boolean existInBackDatabaseSavedForFlushRemove; // 保存上面存在标志，用来提高并发（避免锁）
    //      记录是否在后端数据库中存在，是为了优化记录不存在时，避免把删除操作发送给后端数据库。
    //      一般后端数据库删除不存在的记录的操作代价和删除存在是一样的。这种情况的案例：
    //      如果应用创建了很多临时记录，但是还没有提交到后端数据库前就删除了。如果没有这个标志，
    //      Zeze会这种情况下的把删除操作交给后端数据库，造成性能浪费。

  - e) ConcurrentLruLike 的辅助变量

    volatile ConcurrentHashMap<K, Record1<K, V>> lruNode;

    Zeze.Util.ConcurrentLruLike 是一个通用的包装。
    Zeze.Transaction.TableCache 也类似ConcurrentLruLike，但是专用的，可以省一点内存和提高一点性能。

    java 默认的lru并发性能不够好，所以写了这个ConcurrentLruLike，它实际的并发特性来源自ConcurrentHashMap。
    简单说明一下：
    ConcurrentHashMap<K, Record1<K, V>> dataMap;                         // Lru里面所有的记录映射。
    ConcurrentLinkedQueue<ConcurrentHashMap<K, Record1<K, V>>> lruQueue; // 记录活跃时所属的映射的队列，按时间排序。
    ConcurrentHashMap<K, Record1<K, V>> lruHot;                          // 热点映射，肯定不会被回收。
    java的lru用LinkedList结构记录了全访问顺序队列，这造成它的并发性能不高。
    ConcurrentLruLike基于ConcurrentHashMap，提供修改的并发访问，访问顺序队列变得不精确，按lruQueue的精度推进。


## 乐观锁详解

乐观锁详解:

1. 排序加锁，实际上所访问的记录存储在SortedDictionary中。
2. 加锁后检查冲突，即数据是否改变。冲突则重做事务。
3. 冲突重做时保持已经得到的锁，这样在冲突非常严重时，第二次执行事务一般都能成功， 
而不会陷入一直冲突，事务永远没法完成的情况。 
4. 重做保持锁，但重做过程中所访问的记录可能发生变化，所以重做仍然需要再次执行lockAndCheck逻辑，
并且处理所访问的记录发生变更的问题。
5. 逻辑处理返回错误码或者异常时也需要检查lockAndCheck，因为乐观锁在实际处理逻辑时没有加锁， 
可能存在并发原因导致本来不应该发生逻辑错误，此时的仍然需要加锁并完成冲突检查，如果冲突了，
也需要重做。

```java
public long perform(Procedure procedure) {
    try { // 【7.】 try 1. 释放记录锁
        var checkpoint = procedure.getZeze().getCheckpoint();
        if (checkpoint == null)
            return Procedure.Closed;
        for (int tryCount = 0; tryCount < 256; ++tryCount) { // 【6.】 最多尝试次数
            // 默认在锁内重复尝试，除非CheckResult.RedoAndReleaseLock，否则由于CheckResult.Redo保持锁会导致死锁。
            checkpoint.enterFlushReadLock(); // 【8.】 对于启用了Global的分布式事务，这个函数是空的。
            try { // 【7.】 try 2. 释放checkpoint读锁，当发生RedoAndReleaseLock时，需要全部释放锁，包括这把锁。
                for (; tryCount < 256; ++tryCount) { // 【6.】 最多尝试次数
                    CheckResult checkResult = CheckResult.Redo; // 用来决定是否释放锁，除非 _lock_and_check_ 明确返回需要释放锁，否则都不释放。
                    try { // 【7.】 try 3. 锁和重做
                        var result = procedure.call();
                        switch (state) { // 【11.】state 控制事务的执行状态，当重要的状态变化的时候，马上会在触发点修改state，并最终在这里检查。
                        // 避免逻辑流程捕捉了重要异常，而丢失掉状态。这是以前xdb不能捕捉Error的原因。现在有了这个状态，即使逻辑流程吃掉了一些异常，
                        // 执行流程仍然不会丢失状态控制。
                        case Running:
                            var saveSize = savepoints.size();
                            if ((result == Procedure.Success && saveSize != 1)
                                    || (result != Procedure.Success && saveSize > 0)) {
                                // 这个错误不应该重做
                                logger.fatal("Transaction.Perform:{}. savepoints.Count != 1.", procedure);
                                finalRollback(procedure);
                                return Procedure.ErrorSavepoint;
                            }
                            checkResult = lockAndCheck(procedure.getTransactionLevel()); // 【10.】不管procedure.call返回什么都需要加锁检查冲突，参见上面的第5点。
                            if (checkResult == CheckResult.Success) {
                                if (result == Procedure.Success) {
                                    finalCommit(procedure); // 【9.】事务成功最终提交处理。
                                    // 正常一次成功的不统计，用来观察redo多不多。
                                    // 失败在 Procedure.cs 中的统计。
                                    if (tryCount > 0) {
                                        if (Macro.enableStatistics) {
                                            ProcedureStatistics.getInstance().getOrAdd("Zeze.Transaction.TryCount").getOrAdd(tryCount).increment();
                                        }
                                    }
                                    return Procedure.Success;
                                }
                                finalRollback(procedure, true);
                                return result;
                            }
                            break; // retry

                        case Abort:
                            logger.warn("Transaction.Perform: Abort");
                            finalRollback(procedure);
                            return Procedure.AbortException;

                        case Redo:
                            //checkResult = CheckResult.Redo;
                            break; // retry

                        case RedoAndReleaseLock:
                            checkResult = CheckResult.RedoAndReleaseLock;
                            break; // retry
                        }
                        // retry clear in finally
                        if (alwaysReleaseLockWhenRedo && checkResult == CheckResult.Redo)
                            checkResult = CheckResult.RedoAndReleaseLock;
                        triggerRedoActions(); // 【11.】触发Redo。
                    } catch (Throwable e) {
                        // Procedure.Call 里面已经处理了异常。只有 unit test 或者重做或者内部错误会到达这里。
                        // 在 unit test 下，异常日志会被记录两次。
                        switch (state) {
                        case Running:
                            logger.error("Transaction.Perform:{} exception. run count:{}", procedure, tryCount, e);
                            if (!savepoints.isEmpty()) {
                                // 这个错误不应该重做
                                logger.fatal("Transaction.Perform:{}. exception. savepoints.Count != 0.", procedure, e);
                                finalRollback(procedure);
                                return Procedure.ErrorSavepoint;
                            }
                            // 对于 unit test 的异常特殊处理，与unit test框架能搭配工作
                            if (e instanceof AssertionError) {
                                finalRollback(procedure);
                                throw (AssertionError)e;
                            }
                            checkResult = lockAndCheck(procedure.getTransactionLevel()); // 【10.】异常也需要检查冲突，参见上面的第5点。
                            if (checkResult == CheckResult.Success) {
                                finalRollback(procedure, true);
                                return Procedure.Exception;
                            }
                            // retry
                            break;

                        case Abort:
                            if (!"GlobalAgent.Acquire Failed".equals(e.getMessage()) &&
                                    !"GlobalAgent In FastErrorPeriod".equals(e.getMessage()))
                                logger.warn("Transaction.Perform: Abort", e);
                            finalRollback(procedure);
                            return Procedure.AbortException;

                        case Redo:
                            checkResult = CheckResult.Redo;
                            break;

                        case RedoAndReleaseLock:
                            checkResult = CheckResult.RedoAndReleaseLock;
                            break;

                        default: // case Completed:
                            if (e instanceof AssertionError)
                                throw (AssertionError)e;
                        }
                        triggerRedoActions();
                        // retry
                    } finally {
                        if (checkResult == CheckResult.RedoAndReleaseLock) {
                            holdLocks.forEach(Lockey::exitLock);
                            holdLocks.clear();
                        }
                        // retry 可能保持已有的锁，清除记录和保存点。
                        accessedRecords.clear();
                        savepoints.clear();
                        actions.clear();
                        redoActions.clear();

                        state = TransactionState.Running; // prepare to retry
                    }

                    if (checkResult == CheckResult.RedoAndReleaseLock) {
                        // logger.debug("CheckResult.RedoAndReleaseLock break {}", procedure);
                        break; // 【12.】需要释放锁重做，跳出这一层循环。
                    }
                }
            } finally {
                checkpoint.exitFlushReadLock();
            }
            //logger.Debug("Checkpoint.WaitRun {0}", procedure);
            // 实现Fresh队列以后删除Sleep。// 【13.】避免忙等。
            try {
                Thread.sleep(Zeze.Util.Random.getInstance().nextInt(80) + 20);
            } catch (InterruptedException e) {
                logger.error("", e);
            }
        }
        logger.error("Transaction.Perform:{}. too many try.", procedure);
        finalRollback(procedure);
        return Procedure.TooManyTry;
    } finally {
        holdLocks.forEach(Lockey::exitLock);
        holdLocks.clear();
    }
}

private CheckResult lockAndCheck(TransactionLevel level) {
    boolean allRead = true;
    var saveSize = savepoints.size();
    if (saveSize > 0) {
        // 全部 Rollback 时 Count 为 0；最后提交时 Count 必须为 1；
        // 其他情况属于Begin,Commit,Rollback不匹配。外面检查。
        // 【扫描修改日志，设置记录Dirty标志，控制读写锁】
        var it = savepoints.get(saveSize - 1).logIterator();
        if (it != null) {
            while (it.moveToNext()) {
                // 特殊日志。不是 bean 的修改日志，当然也不会修改 Record。
                // 现在不会有这种情况，保留给未来扩展需要。
                Log log = it.value();
                if (log.getBean() == null)
                    continue;

                TableKey tkey = log.getBean().tableKey();
                var record = accessedRecords.get(tkey);
                if (record != null) {
                    record.dirty = true;
                    allRead = false;
                } else {
                    // 只有测试代码会把非 Managed 的 Bean 的日志加进来。
                    logger.fatal("impossible! record not found.");
                }
            }
        }
    }

    if (allRead && level == TransactionLevel.AllowDirtyWhenAllRead)
        return CheckResult.Success; // 使用一个新的enum表示一下？

    boolean conflict = false; // 冲突了，也继续加锁，为重做做准备！！！
    if (holdLocks.isEmpty()) {
        // 【优化】第一次加锁，按顺序即可。
        for (var e : accessedRecords.entrySet()) {
            var r = lockAndCheck(e);
            switch (r) {
            case Success:
                break;
            case Redo:
                conflict = true;
                break; // continue lock
            default:
                return r;
            }
        }
        return conflict ? CheckResult.Redo : CheckResult.Success;
    }

    // 【两个排序好的队列：A. 访问的记录，B. 已经持有的锁，】
    int index = 0;
    int n = holdLocks.size();
    final var ite = accessedRecords.entrySet().iterator();
    var e = ite.hasNext() ? ite.next() : null;
    while (null != e) {
        // 如果 holdLocks 全部被对比完毕，直接锁定它
        if (index >= n) {
            var r = lockAndCheck(e);
            switch (r) {
            case Success:
                break;
            case Redo:
                conflict = true;
                break; // continue lock
            default:
                return r;
            }
            e = ite.hasNext() ? ite.next() : null;
            continue;
        }

        Lockey curLock = holdLocks.get(index);
        int c = curLock.getTableKey().compareTo(e.getKey());

        // holdLocks a  b  ...
        // needLocks a  b  ...
        if (c == 0) {
            // 这里可能发生读写锁提升
            if (e.getValue().dirty && !curLock.isWriteLockHeld()) {
                // 必须先全部释放，再升级当前记录锁，再锁后面的记录。
                // 直接 unlockRead，lockWrite会死锁。
                n = _unlock_start_(index, n);
                // 从当前index之后都是新加锁，并且index和n都不会再发生变化。
                // 重新从当前 e 继续锁。
                continue;
            }
            // BUG 即使锁内。Record.Global.State 可能没有提升到需要水平。需要重新_check_。
            var r = _check_(e.getValue().dirty, e.getValue());
            switch (r) {
            case Success:
                // 已经锁内，所以肯定不会冲突，多数情况是这个。
                break;
            case Redo:
                // Impossible!
                conflict = true;
                break; // continue lock
            default:
                // _check_可能需要到Global提升状态，这里可能发生GLOBAL-DEAD-LOCK。
                return r;
            }
            ++index;
            e = ite.hasNext() ? ite.next() : null;
            continue;
        }
        // holdLocks a  b  ...
        // needLocks a  c  ...
        if (c < 0) {
            // 释放掉 比当前锁序小的锁，因为当前事务中不再需要这些锁
            int unlockEndIndex = index;
            for (; unlockEndIndex < n && holdLocks.get(unlockEndIndex).getTableKey().compareTo(e.getKey()) < 0; ++unlockEndIndex)
                holdLocks.get(unlockEndIndex).exitLock();
            holdLocks.subList(index, unlockEndIndex).clear();
            n = holdLocks.size();
            // 重新从当前 e 继续锁。
            continue;
        }

        // holdLocks a  c  ...
        // needLocks a  b  ...
        // 为了不违背锁序，释放从当前锁开始的所有锁
        n = _unlock_start_(index, n);
        // 重新从当前 e 继续锁。
    }
    return conflict ? CheckResult.Redo : CheckResult.Success;
}

private CheckResult lockAndCheck(Map.Entry<TableKey, RecordAccessed> e) {
    Lockey lockey = locks.get(e.getKey());
    boolean writeLock = e.getValue().dirty;
    lockey.enterLock(writeLock);
    holdLocks.add(lockey);
    return _check_(writeLock, e.getValue());
}

private static CheckResult _check_(boolean writeLock, RecordAccessed e) {
    e.atomicTupleRecord.record.enterFairLock();
    try {
        if (writeLock) {
            switch (e.atomicTupleRecord.record.getState()) {
            case GlobalCacheManagerConst.StateRemoved:
                // 被从cache中清除，不持有该记录的Global锁，简单重做即可。
                return CheckResult.Redo;

            case GlobalCacheManagerConst.StateInvalid:
                return CheckResult.RedoAndReleaseLock; // 写锁发现Invalid，可能有Reduce请求。

            case GlobalCacheManagerConst.StateModify:
                return e.atomicTupleRecord.timestamp != e.atomicTupleRecord.record.getTimestamp() ? CheckResult.Redo : CheckResult.Success;

            case GlobalCacheManagerConst.StateShare:
                // 这里可能死锁：另一个先获得提升的请求要求本机Reduce，但是本机Checkpoint无法进行下去，被当前事务挡住了。
                // 通过 GlobalCacheManager 检查死锁，返回失败;需要重做并释放锁。
                var acquire = e.atomicTupleRecord.record.acquire(GlobalCacheManagerConst.StateModify,
                        e.atomicTupleRecord.record.isFresh(), false);
                if (acquire.resultState != GlobalCacheManagerConst.StateModify) {
                    e.atomicTupleRecord.record.setNotFresh(); // 抢失败不再新鲜。
                    logger.debug("Acquire Failed. Maybe DeadLock Found {}", e.atomicTupleRecord);
                    e.atomicTupleRecord.record.setState(GlobalCacheManagerConst.StateInvalid); // 这里保留StateShare更好吗？
                    return CheckResult.RedoAndReleaseLock;
                }
                e.atomicTupleRecord.record.setState(GlobalCacheManagerConst.StateModify);
                return e.atomicTupleRecord.timestamp != e.atomicTupleRecord.record.getTimestamp() ? CheckResult.Redo : CheckResult.Success;
            }
            return e.atomicTupleRecord.timestamp != e.atomicTupleRecord.record.getTimestamp() ? CheckResult.Redo : CheckResult.Success; // impossible
        }
        switch (e.atomicTupleRecord.record.getState()) {
        case GlobalCacheManagerConst.StateRemoved:
            // 被从cache中清除，不持有该记录的Global锁，简单重做即可。
            return CheckResult.Redo;

        case GlobalCacheManagerConst.StateInvalid:
            return CheckResult.RedoAndReleaseLock; // 发现Invalid，可能有Reduce请求或者被Cache清理，此时保险起见释放锁。
        }
        return e.atomicTupleRecord.timestamp != e.atomicTupleRecord.record.getTimestamp() ? CheckResult.Redo : CheckResult.Success;
    } finally {
        e.atomicTupleRecord.record.exitFairLock();
    }
}
```

## 单点的Global

1. 介绍
多台服务器共享后台数据库。每台服务器拥有自己的缓存。一致性缓存就是维护多台服务器之间缓存的一致性。
zeze一致性缓存和CPU-Cache-Memory的结构很像。所以参考了CPU的MESI协议自己实现了一个锁分配机制。
这个一致性缓存思路是非常暴力的，核心出发点就是Global是全能的，知道所有东西。所以算法也非常直接简单。
使用了类似MESI的状态名，记录分成读写不可用三种状态。不可用表示记录本地还没有权限，读状态允许同时存在于
多台服务器缓存中，写状态只允许在一台服务器中。在一致性缓存之上，每一台服务器的事务就能像自己独占所有
数据一样，完成本地事务即可。这就是基于一致性缓存的分布式事务。

2. 基本流程
记录的三个锁状态：Modify,Share,Invalid。
当主逻辑服务器需要访问或修改数据时，向全局锁管理服务器（下面用Global称呼）申请Modify或Share锁。
Global知道所有记录的锁的分布状态。它根据申请的锁，向现拥有者发送相应的降级请求；拥有者释放锁，
并把数据刷新到后端服务器后，才给Global返回结果；Global登记申请者的锁状态，给申请者返回结果。

3. 并发原则
对于同一个记录，所有的申请请求是排队处理的，一个时候仅处理一个请求。根据请求是读或写，按读写共享
互斥规则完成锁的分配管理。
对于不同的记录，申请执行是并发的。互相不会影响。

4. 死锁
由于每个服务器都采取了排序加锁（乐观锁），所以多个记录间已经不会死锁了。
但Global对同一个记录的访问，即使只有一个记录也是有可能死锁的。比如：
ServerA 拥有记录1的读锁，然后去申请写锁。
ServerB 也拥有记录1的读锁，然后去申请写锁。
当Global在处理ServerA的请求时需要通知ServerB降级，但ServerB此时也在等待写锁申请，
此时ServerB等待记录1的Global写锁时，已经在本进程占有了记录1的锁，降级请求得不到执行。
如果没有一个检测机制打破这个循环，就死锁了。
这个叫一个记录的死锁。

5. 并发实现伪码
```java
int acquireShare(Acquire rpc) {
    while (true) {
        // 查询得到记录。
        CacheState cs = global.computeIfAbsent(rpc.Argument.globalKey, CacheState::new);

        // 锁住这个记录。
        synchronized (cs) {
            // 记录状态是被删除的，继续循环，获取新的记录。
            // 这是因为computeIfAbsent和synchronized (cs)之间存在时间窗口，有可能在这个窗口记录刚好被删除了。
            // 这里继续循环，看作第一次申请这个记录的锁即可。
            // 这里的关键是，所有的同一个globalKey的请求，都必须访问同一个CacheState。
            if (cs.acquireStatePending == StateRemoved)
                continue;

            while (cs.acquireStatePending != StateInvalid && cs.acquireStatePending != StateRemoved) {
                if (检测到死锁()) {
                    // 死锁，发送申请失败结果
                    rpc.Result.state = StateInvalid;
                    rpc.SendResultCode(AcquireShareDeadLockFound);
                    return 0;
                }
                cs.wait(); //await 等通知！这个CacheState有进行中的请求。等待完成。这里就是并发排队。
                // 为什么这里的并发访问不能简单的通过synchronized (cs)就是先互斥和等待，而要用cs.wait()呢？
                // 这是为了上面的死锁检测，如果仅仅通过锁互斥，那么等待的请求进不来，而进行总的请求又在等待
                // 正在申请的，就无法完成死锁检查，那么死锁发生的时候，就没法解锁了。使用cs.wait()，后面
                // 进行中的请求在等待操作的时候，也会进行cs.wait()，这样就能开门放狗，把当前所有的请求者都
                // 放进来进行死锁检测，最终死锁才可能被打断。
            }
            if (cs.acquireStatePending == StateRemoved)
                continue; // concurrent release，同上一个StateRemoved判断。

            // 获得执行权限，首先设置申请状态。
            cs.acquireStatePending = StateShare;

            if (cs.modify != null) {
                // 如果当前cs的是写状态，需要对当前锁拥有者进行降级处理。
                cs.modify.reduce(() -> {
                    // 对当前写拥有者发起异步降级操作，操作成功回调这个lambda。
                    // ...
                    // 得到结果，发送notifyAll。
                    synchronized (cs) {
                        cs.notifyAll(); //notify 通知所有排队的申请者进来 (line a)
                    }
                });

                cs.wait(); // 等待reduce结果，
                // 这里进入wait后，如果存在排队的申请者，就可以获得锁，进到死锁检测循环哪里。
                // 当上面的(line a)行得到降级结果发起通知时，所有的等待的都会唤醒，包括这里。
                // 没有获得执行权的会再次等待在死锁检测循环哪个cs.wait()哪里。
                // 这个唤醒以后会继续完成自己的申请流程。

                // 处理锁分配相关状态修改；
                sender.acquired.put(gKey, StateShare);
                cs.modify = null;
                cs.share.add(sender);

                // 释放执行状态；
                cs.acquireStatePending = StateInvalid;
                cs.notifyAll(); //notify 通知所有排队的申请者进来。

                // 发送结果；
                rpc.SendResultCode(0);
                return 0;
            }

            // 当前拥有者也是拥有读锁，允许共享读取，直接顺序完成下面操作即可。

            // 处理锁分配相关状态修改；
            sender.acquired.put(gKey, StateShare);
            cs.share.add(sender);

            // 释放执行状态；
            cs.acquireStatePending = StateInvalid;
            cs.notifyAll(); //notify 通知所有排队的申请者进来。

            // 发送结果；
            rpc.SendResultCode(0);
            return 0;
        }
    }
}
```

为什么是 cs.notifyAll() ?

因为死锁检测要求所有等待的申请者都轮流检查一遍，如果仅仅放一个进来，那么如果放进来这个会发生死锁，
而被依赖等待的申请者又在排队，那么死锁检测就会失败。所以每次都需要全部唤醒。

6. Global单点问题的解决方案
Global从结构和算法上是单点的。系统规模很大的时候，单个进程性能会有上限。这个问题的解决方案运行多个Global。
GlobalIndex = hash(GlobalTableKey) % GlobalInstanceCount; 把记录的请求分不到多个Global实例中，根据上面第3点的
并发规则，这样划分请求是正确的，并且可以把负载分不开。

7. Global单点可靠性的解决方案
Global是单点的，如果出现进程或者机器异常，会导致系统不可用。这个问题的解决方案是引入Raft算法。对于每个Global实例，
都有多个Raft备份，只要Raft系统中的健康节点超过Raft节点数/2个，系统就能持续提供服务。
Raft系统的问题是性能会下降很多。具体应用需要根据自己的需求，决定是否采用Global Raft版本。

## 关联记录集合

	一个事务中修改的记录集合就是一个关联记录集合。只要整个关联记录集合一起按事务的方式保存到后端数据库，事务就被保证了。
	考虑几个问题：1. 事务需要能并发的执行的问题；2. 事务是需要系列化的，有时序的问题；3. 事务保存到后端数据库的并发问题；
	
	1. 最简单的方案：只有一个唯一的保存事务队列，事务提交的时候就加入队列，这样上面的问题1，2就解决了，问题3被忽略了。

	2. 很早以前就有初步想法，当时每个事务一个关联集合，不敢马上合并，觉得效率不够。这样的事务的关联集合有可能有交集，所以
	最终并发保存的时候需要有时序，想了一个串串的思路。每个记录一根竖线，每个事务操作的记录横向连成一行珠子，整个横向珠子
	沿竖线往下落，某个记录竖线上面存在多个珠子时，根据先后关系自动形成顺序，并发保存从最下面提取横向的珠子，可以同时
	提取出来的可以并发保存，竖线上叠起来的按时序保存。这个思路的并发保存方案不是很好的实现。实际上我还没想过是否存在一个
	简洁的方法实现这个思路。说这个仅仅是，这个还有点意思。

	| | | | | |
	| +-+-+ | |
	| | | +-+-+
	1 2 3 4 5 6

	3. 重新考虑这个需求时，按照新的暴力原则，在事务提交的时候，马上把有交集的关联记录集合合并起来，这样存在的
	关联记录集合都是没有交集的，并发保存变得非常简单，结构也清晰。这就是Zeze的当前方案，这个方案事务提交会慢一些。

	伪码
	void tryUpdateAndCheckpoint(Transaction trans) {
		var transAccessRecords = new HashSet<Record>(); // 当前事务的所有访问的记录
		var relativeRecordSets = new TreeMap<Long, RelativeRecordSet>(); // 需要按关联记录集合的编号排序。
		for (var ar : trans.getAccessedRecords().values()) {
			var record = ar.atomicTupleRecord.record;
			transAccessRecords.add(record);
			relativeRecordSets.put(volatileRrs.id, volatileRrs); // 搜集当前事务中记录的关联记录集合。
		}

		var lockedRelativeRecordSets = new ArrayList<RelativeRecordSet>();
		try {
			// 锁住所有当前的关联记录集合。这个锁定算法和乐观锁算法基本原则是一样的。
			_lock_(lockedRelativeRecordSets, relativeRecordSets, transAccessRecords);
			if (!lockedRelativeRecordSets.isEmpty()) {
				var mergedSet = _merge_(lockedRelativeRecordSets, trans, allRead);
				// 根据配置，马上保存，或者加到总的关联记录集合容器中等待定时保存
				flush(mergedSet) or relativeRecordSetMap.add(mergedSet);
			}
		} finally {
			lockedRelativeRecordSets.forEach(RelativeRecordSet::unLock);
		}
	}

## Listener
订阅数据变更。

实现历史:

1. Xdb时代，在脏记录通告的流程中同时收集变更信息，造成Listener的实现代码和核心流程耦合起来，非常不好理解。
2. Zeze的Listener第二版，采取完全独立的算法和规则，跟正常的逻辑分开，变得更清晰了。而且效率不差。
3. Zeze的Lisnter当前版，算法和规则还是独立的，但是实现被分散到不同的修改日志类。这个版本实际上没有第二版清晰。
   但实现了以后，带来一个通用的增量同步数据的能力。这个版本一开始是为RocksRaft写的，RocksRaft需要增量同步数据的能力。
   后来觉得这个能力挺不错的，就把Zeze的Listener也改成这个方式了。


## RocksDb 误打误撞

接入Java RocksDb时，时间紧迫，不熟悉包装，沿用了c#的FamilyColumn的机制来在一个Db内实现多张表。
带来的好处是整个应用一个RocksDb实例，使用Batch，没有使用事务。作为不需要扩展的系统，凑合了。
