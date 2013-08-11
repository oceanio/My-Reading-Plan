Notes on “MongoDB Applied Design Patterns”
===========================

### Chapter 1 To Embed or Reference  

* 是否采用多级结构化，还是关系数据库的引用关系方式，是一个需要权衡的问题。

### Chapter 2 Polymorphic Schemas

* 当一个collection中的文档结构相似，但不完全一样，我们称为Polymorphic Schema

### Chapter 3 Mimicking Transactional Behavior

* MongoDB不支持事务，所以多文档操作有异常的可能，所以如果需要事务保护，需要自己实现（比较弱）  

Transaction的文档结构设计如下：
  
    {
        _id: ObjectId(...),
        state: 'new',
        ts: ISODateTime(...),
        amt: 55.22,
        src: 1,
        dst: 2
    }
    
account的文档结构设计如下：
 
    { _id: 1, balance: 100, txns: [] }
    { _id: 2, balance: 0, txns: [] }
    
转账的事务实现如下：
    
    def transfer(amt, source, destination, max_txn_time):
        txn = prepare_transfer(amt, source, destination)
        commit_transfer(txn, max_txn_time)
        
    def prepare_transfer(amt, source, destination):
        # Create a transaction object
        now = datetime.utcnow()
        txnid = ObjectId()
        txn = {
            '_id': txnid,
            'state': 'new',
            'ts': datetime.utcnow(),
            'amt': amt,
            'src': source,
            'dst': destination }
        db.transactions.insert(txn)
        
        # "Prepare" the accounts
        result = db.accounts.update(
            { '_id': source, 'balance': { '$gte': amt } },
            { '$inc': { 'balance': -amt },
              '$push': { 'txns': txn['_id'] } })
        if not result['updatedExisting']:
            db.transaction.remove({'_id': txnid})
            raise InsufficientFundsError(source)
        db.accounts.update(
            { '_id': dest },
            { '$inc': { 'balance': amt },
              '$push': { 'txns': txn['_id'] } })
        return txn

    def commit_transfer(txn, max_txn_time):
        # Mark the transaction as committed
        now = datetime.utcnow()
        cutoff = now - max_txn_time
        result = db.transaction.update(
            { '_id': txnid, 'state': 'new', 'ts': { '$gt': cutoff } },
            { '$set': { 'state': 'commit' } })
        if not result['updatedExisting']:
            raise TransactionError(txn['_id'])
        else:
            retire_transaction(txn['_id'])
            
    def retire_transaction(txn_id):
        db.accounts.update(
            { '_id': txn['src'], 'txns._id': txn_id },
            { '$pull': { 'txns': txn_id } })
        db.accounts.update(
            { '_id': txn['dst'], 'txns._id': txn['_id'] },
            { '$pull': { 'txns': txn_id } })
        db.transaction.remove({'_id': txn_id})
        
如果上面的操作发生任何错误，定时任务会定时调用下面的代码，清除异常的事务：

    # priodical cleanup task
    def cleanup_transactions(txn, max_txn_time):
        # Find & commit partially-committed transactions
        for txn in db.transaction.find({ 'state': 'commit' }, {'_id': 1}):
            retire_transaction(txn['_id'])
            
        # Move expired transactions to 'rollback' status:
        cutoff = now - max_txn_time
        db.transaction.update(
            { '_id': txnid, 'state': 'new', 'ts': { '$lt': cutoff } },
            { '$set': { 'state': 'rollback' } })
            
        # Actually rollback transactions
        for txn in db.transaction.find({ 'state': 'rollback' }):
            rollback_transfer()
            
    def rollback_transfer(txn):
        db.accounts.update(
            { '_id': txn['src'], 'txns._id': txn['_id'] },
            { '$inc': { 'balance': txn['amt'] },
              '$pull': { 'txns': { '_id': txn['_id'] } } })
        db.accounts.update(
            { '_id': txn['dst'], 'txns._id': txn['_id'] },
            { '$inc': { 'balance': -txn['amt'] },
              '$pull': { 'txns': { '_id': txn['_id'] } } })
        db.transaction.remove({'_id': txn['_id']})

* * *

### Chapter 4 Operational Intelligence

**使用MongoDB存储日志**

By setting `w=0`, you do not require that MongoDB acknowledge receipt of the insert. Although this is the fastest option available to us, it also carries with it the risk that you might lose a large number of events before you notice.  

    db.events.insert(event, w=0)

If you want to ensure that MongoDB acknowledges inserts, you can omit the `w=0` argument, or pass `w=1` (the default) as follows:  

    db.events.insert(event)
    # Alternatively, you can do this
    db.events.insert(event, w=1)

This will force your application to acknowledge that the data has replicated to two members of the replica set. 

    db.events.insert(event, w=2)

**批量插入**

批量插入的问题就是发生任何错误，只有一部分插入，后面的就停止了。如果忽略错误继续插入，使用`continue_on_error=True`参数。这样批量插入可以一直执行，但是如果发生多次失败，只能返回最后一次的错误。


**管理索引**

查看索引的内存，mongoDB要求索引全部加载到内存中，这也是mongoDB吃内存的原因。

    db.command('collstats', 'events')['indexSizes']

索引的顺序很重要，比如：

    db.events.ensure_index([('time', 1), ('host', 1)])

使用`explain()`分析查询效率：

    { ..
        u'cursor': u'BtreeCursor time_1_host_1',
        u'indexBounds': {u'host': [[u'127.0.0.1', u'127.0.0.1']],
            u'time': [
                [ datetime.datetime(2000, 10, 10, 0, 0),
                  datetime.datetime(2000, 10, 11, 0, 0)]]
        },
        ...
        u'millis': 4,
        u'n': 11,
        u'nscanned': 1296,
        u'nscannedObjects': 11,
    ... }

This query had to scan 1,296 items from the index to return 11 objects in 4 milliseconds.

如果调整一下顺序：

    db.events.ensure_index([('host', 1), ('time', 1)])

Now, explain() tells us the following:

    { ...
        u'cursor': u'BtreeCursor host_1_time_1',
        u'indexBounds': {u'host': [[u'127.0.0.1', u'127.0.0.1']],
        u'time': [[datetime.datetime(2000, 10, 10, 0, 0),
            datetime.datetime(2000, 10, 11, 0, 0)]]},
        ...
        u'millis': 0,
        u'n': 11,
        ...
        u'nscanned': 11,
        u'nscannedObjects': 11,
        ...
    }

Here, the query had to scan 11 items from the index before returning 11 objects in less than a millisecond.

**索引的注意点**

* Any fields that will be queried by equality should occur first in the index definition.
* Fields used to sort should occur next in the index definition. If multiple fields are being sorted (such as (last_name, first_name), then they should occur in the same order in the index definition.
* Fields that are queried by range should occur last in the index definition.

* Whenever we have a range query on two or more properties, they cannot both be used effectively in the index.
* Whenever we have a range query combined with a sort on a different property, the index is somewhat less efficient than when doing a range and sort on the same property set.

In such cases, the best approach is to test with representative data, making liberal use of `explain()`. If you discover that the MongoDB query optimizer is making a bad choice of index (perhaps choosing to reduce the number of entries scanned at the expense of doing a large in-memory sort, for instance), you can also use the `hint()` method to tell it which index to use.

**Sharding Concern**

In a sharded environment, the limitations on the maximum insertion rate are:
* The number of shards in the cluster
* The shard key you choose

Ideally, your shard key should have two characteristics:
* Insertions are balanced between shards
* Most queries can be routed to a subset of the shards to be satisfied


+ **Option 1: Shard by time**

    Although using the timestamp, or the ObjectId in the _id field, would distribute your data evenly among shards, these keys lead to two problems:
    * All inserts always flow to the same shard, which means that your shard cluster will have the same write throughput as a standalone instance.
    * Most reads will tend to cluster on the same shard, assuming you access recent data more frequently.

+ **Option 2: Shard by a semi-random key**

    Using this shard key, or any hashed value as a key, presents the following downsides:
    * The shard key, and the index on the key, will consume additional space in the database.
    * Queries, unless they include the shard key itself, must run in parallel on all shards, which may lead to degraded performance.
    
    This might be an acceptable trade-off in some situations. The workload of event logging systems tends to be heavily skewed toward writing; read performance may not be as critical as perfectly balanced write performance.





