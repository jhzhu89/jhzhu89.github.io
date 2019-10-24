---
layout: single
title:  "修复etcd mvcc space exceeded"
categories:
  - etcd
tags:
  - etcd
toc: true
excerpt_separator: <!--more-->
---

最近在搭建并使用kubemark来做测试的过程中，碰到了一个etcd报错"etcdserver: mvcc: database space exceeded"问题，本文记录了出现了这种问题后如何对etcd进行恢复。

<!--more-->

# 问题的产生

该问题发生的场景如下：

一条测试用例一次性创建了10000个k8s service实例，然后检查一些预设的条件。由于一些原因，这条用例卡住了很久，于是我Ctrl-C中断了这条用例，之后尝试使用kubectl来删除还留存的service实例，大概还剩与5000个左右。这时，kubectl也hang住了，进一步查看apiserver日志，发现apiserver不停的打印*etcdserver: mvcc: database space exceeded*。

我们知道etcd是一个支持mvcc的key value数据库，执行删除，实际上是将kv的某个版本加上了删除标记，并不是立即从数据库中清楚这条数据。等到开始执行compaction，才会真正的删除这条数据。具体代码如下，删除其实是加上了tombstone标记，然后UnsafeSeqPut进boltdb。

```go
// mvcc/kvstore_txn.go

func (tw *storeTxnWrite) delete(key []byte, rev revision) {
	ibytes := newRevBytes()
	idxRev := revision{main: tw.beginRev + 1, sub: int64(len(tw.changes))}
	revToBytes(idxRev, ibytes)
	ibytes = appendMarkTombstone(ibytes)

	kv := mvccpb.KeyValue{Key: key}

	d, err := kv.Marshal()
	if err != nil {
		plog.Fatalf("cannot marshal event: %v", err)
	}

	tw.tx.UnsafeSeqPut(keyBucketName, ibytes, d)
	err = tw.s.kvindex.Tombstone(key, idxRev)
	if err != nil {
		plog.Fatalf("cannot tombstone an existing key (%s): %v", string(key), err)
	}
	tw.changes = append(tw.changes, kv)

	item := lease.LeaseItem{Key: string(key)}
	leaseID := tw.s.le.GetLease(item)

	if leaseID != lease.NoLease {
		err = tw.s.le.Detach(leaseID, []lease.LeaseItem{item})
		if err != nil {
			plog.Errorf("cannot detach %v", err)
		}
	}
}
```

这条错误，是说etcd的后端存储已经超过了预设的大小，它默认是2G，而我并没有修改过这个设置，所以，这个时间点我的etcd存储容量已经超过了2G，etcd进入了只读模式，并且一直在报警，这时apiserver已经无法继续写入数据了。

# 如何解决

etcd实际上存储了kv的多个版本，所以当发生这个错误的时候，我们可以对etcd做compaction，来清除老的版本。更近一步，我们还可以对etcd进行碎片整理，让boltdb中的数据更紧凑。

## Compaction

### 手动

通过etcdctl 

```bash
ETCDCTL_API=3 etcdctl compact <rev>
```

### 自动

给etcd配置自动进行compaction的周期

```bash
--auto-compaction-retention=1  # 表示每1h做一次 compaction
```

## Defrag

通过etcdctl来做碎片整理

```bash
ETCDCTL_API=3 etcdctl defrag
```

## 恢复步骤

```bash
# step 1: get current revision
$ rev=$(ETCDCTL_API=3 etcdctl --endpoints=:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
# step 2: compact away all old revisions
$ ETCDCTL_API=3 etcdctl compact $rev
compacted revision 1516
# step 3: defragment away excessive space
$ ETCDCTL_API=3 etcdctl defrag
Finished defragmenting etcd member[127.0.0.1:2379]
# step 4: disarm alarm
$ ETCDCTL_API=3 etcdctl alarm disarm
memberID:13803658152347727308 alarm:NOSPACE
# step 5: test puts are allowed again
$ ETCDCTL_API=3 etcdctl put newkey 123
OK
```

## 自定义空间配额

前面提到，etcd默认的大小配置是2GB。这个有一定的历史原因，以前的版本，boltdb超过2GB后可能性能下降较为明显。而在最近的etcd版本中（3.3.15和3.4.0），合并了一个优化boltdb中页分配的算法，据称数据库大小超过100GB也不会有很大的性能下降。我们可以适当的将默认配置调大一些，来减少发生这个错误的概率。这个配置参数如下。

```bash
--quota-backend-bytes=8589934592 # 8GB
```
