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

#### 4.1 使用MongoDB存储日志

+ **插入效率**

    By setting `w=0`, you do not require that MongoDB acknowledge receipt of the insert. Although this is the fastest option available to us, it also carries with it the risk that you might lose a large number of events before you notice.  

        db.events.insert(event, w=0)

    If you want to ensure that MongoDB acknowledges inserts, you can omit the `w=0` argument, or pass `w=1` (the default) as follows:  

        db.events.insert(event)
        # Alternatively, you can do this
        db.events.insert(event, w=1)

    This will force your application to acknowledge that the data has replicated to two members of the replica set. 

        db.events.insert(event, w=2)

+ **批量插入**

    批量插入的问题就是发生任何错误，只有一部分插入，后面的就停止了。如果忽略错误继续插入，使用`continue_on_error=True`参数。这样批量插入可以一直执行，但是如果发生多次失败，只能返回最后一次的错误。


+ **管理索引**

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

+ **索引的注意点**

    * Any fields that will be queried by equality should occur first in the index definition.
    * Fields used to sort should occur next in the index definition. If multiple fields are being sorted (such as (last_name, first_name), then they should occur in the same order in the index definition.
    * Fields that are queried by range should occur last in the index definition.

    * Whenever we have a range query on two or more properties, they cannot both be used effectively in the index.
    * Whenever we have a range query combined with a sort on a different property, the index is somewhat less efficient than when doing a range and sort on the same property set.

    In such cases, the best approach is to test with representative data, making liberal use of `explain()`. If you discover that the MongoDB query optimizer is making a bad choice of index (perhaps choosing to reduce the number of entries scanned at the expense of doing a large in-memory sort, for instance), you can also use the `hint()` method to tell it which index to use.

+ **Sharding Concern**

    In a sharded environment, the limitations on the maximum insertion rate are:
    * The number of shards in the cluster
    * The shard key you choose

    Ideally, your shard key should have two characteristics:
    * Insertions are balanced between shards
    * Most queries can be routed to a subset of the shards to be satisfied

    下面是几种常用的shard key: 

    + **Option 1: Shard by time**

        Although using the timestamp, or the ObjectId in the _id field, would distribute your data evenly among shards, these keys lead to two problems:
        * All inserts always flow to the same shard, which means that your shard cluster will have the same write throughput as a standalone instance.
        * Most reads will tend to cluster on the same shard, assuming you access recent data more frequently.

    + **Option 2: Shard by a semi-random key**

        Using this shard key, or any hashed value as a key, presents the following downsides:
        * The shard key, and the index on the key, will consume additional space in the database.
        * Queries, unless they include the shard key itself, must run in parallel on all shards, which may lead to degraded performance.
        
        This might be an acceptable trade-off in some situations. The workload of event logging systems tends to be heavily skewed toward writing; read performance may not be as critical as perfectly balanced write performance.

    + **Option 3: Shard by an evenly distributed key in the data set**

        针对http日志这个例子, you might consider using the *path* field. This has a couple of advantages:
        * Writes will tend to balance evenly among shards.
        * Reads will tend to be selective and local to a single shard if the query selects on the path field.

        The biggest potential drawback to this approach is that all hits to a particular path must go to the same chunk, and that chunk cannot be split by MongoDB, since all the documents in it have the same shard key. This might not be a problem if you have fairly even load on your website, but if one page gets a disproportionate number of hits, you can end up with a large chunk that is completely unsplittable that causes an unbalanced load on one shard.

    + **Option 4: Shard by combining a natural and synthetic key**

        MongoDB supports compound shard keys that combine the best aspects of options 2 and 3. In these situations, the shard key would resemble `{ path: 1 , ssk: 1 }`, where path is an often-used natural key or value from your data and ssk is a hash of the `_id` field.

        In most situations, these kinds of keys provide the ideal balance between distributing writes across the cluster and ensuring that most queries will only need to access a select number of shards.

    Selecting shard keys is difficult because there are no definitive “best practices,” the decision has a large impact on performance, and it is difficult or impossible to change the shard key after making the selection. The best solution is to test with your own data.

+ **管理数据库大小**

    + **Capped collections**

        Capped collections have a fixed size, and drop old data automatically when inserting new data after reaching cap. 但是在现在的版本中，it is not possible to shard capped collections.

    + **TTL collections**

        If you want something like capped collections that can be sharded, you might consider using a “time to live” (TTL) index on that collection.

            db.events.ensureIndex('time', expireAfterSeconds=3600)

        缺点是，`remove()`方法性能没有优化，而且删除可能导致fragmentation

    + **Multiple collections, single database**

        类似于rotate logfile的方法，rotate数据表

        This approach has several advantages over the single collection approach:
        * Collection renames are fast and atomic.
        * MongoDB does not bring any documents into memory to drop a collection.
        * MongoDB can effectively reuse space freed by removing entire collections without leading to data fragmentation.

        Nevertheless, this operation may increase some complexity for queries, if any of your analyses depend on events that may reside in the current and previous collection. For most real-time data-collection systems, this approach is ideal.

    + **Multiple databases**

        同上，但是rotate数据库。缺点是程序会非常复杂。

#### 4.2 预处理报告

+ **One document per page per day, flat documents**

    This approach has a couple of advantages:
    * For every request on the website, you only need to update one document.
    * Reports for time periods within the day, for a single page, require fetching a single document.

    当文件增长后，MongoDB会不断的重新分配文件空间，会导致遇到后面性能越差。解决方案就是预分配，把所有字段都用0填充。另外，比如下面的minute字段，当接近12点时，update性能会变差，因为查找时间变长。

        {
            _id: "20101010/site-1/apache_pb.gif",
            metadata: {
                date: ISODate("2000-10-10T00:00:00Z"),
                site: "site-1",
                page: "/apache_pb.gif" },
            daily: 5468426,
            hourly: {
                "0": 227850,
                "1": 210231,
                ...
                "23": 20457 },
            minute: {
                "0": 3612,
                "1": 3241,
                ...
                "1439": 2819 }
        }

+ **One document per page per day, hierarchical documents**

    接近这个问题，就是把minute按照小时进行分段：

        minute: {
            "0": {
                "0": 3612,
                "1": 3241,
                ...
                "59": 2130 },
            "1": {
                "60": ... ,
            },
            ...
            "23": {
                ...
                "1439": 2819 }
        }

+ **Separate documents by granularity level**

    把每天的总数单独拿出来，按月份拆分成另外一个文档。这样打点记录时会调用两次update操作，但是按月查询时会提高效率。

+ **预分配的技巧**

    While we could pre-allocate the documents all at once, this leads to poor performance during the pre-allocation time. A better solution is to pre-allocate the documents probabilistically each time we log a hit:

        from random import random
        from datetime import datetime, timedelta, time
        
        # Example probability based on 500k hits per day per page
        prob_preallocate = 1.0 / 500000
        
        def log_hit(db, dt_utc, site, page):
            if random.random() < prob_preallocate:
            preallocate(db, dt_utc + timedelta(days=1), site_page)
            # Update daily stats doc
            ...

#### 4.3 Hierarchical Aggregation

层级汇总，从raw data定时运行汇总任务，定时计算小时汇总，然后定时计算天的汇总，等等。

`last_run`和`cutoff`两个变量防止重复统计，这样mapreduce可以任意时间运行任意多次（发生了异常怎么办？会重复统计）

    cutoff = datetime.utcnow() - timedelta(seconds=60)

    # First step is still special
    query = { 'ts': { '$gt': last_run, '$lt': cutoff } }
    db.events.map_reduce(
        map=mapf_hour, reduce=reducef,
        finalize=finalizef, query=query,
        out={ 'reduce': 'stats.hourly' })

    # But the other ones are not
    h_aggregate(db.stats.hourly, db.stats.daily, mapf_day, cutoff, last_run)
    h_aggregate(db.stats.daily, db.stats.weekly, mapf_week, cutoff, last_run)
    h_aggregate(db.stats.daily, db.stats.monthly, mapf_month, cutoff, last_run)
    h_aggregate(db.stats.monthly, db.stats.yearly, mapf_year, cutoff, last_run)
    
    last_run = cutoff

`mapf_hour`函数实现如下：

    mapf_hour = bson.Code('''function() {
        var key = {
            u: this.userid,
            d: new Date(
                this.ts.getFullYear(),
                this.ts.getMonth(),
                this.ts.getDate(),
                this.ts.getHours(),
                0, 0, 0);
        };

        emit(
            key,
            {
                total: this.length,
                count: 1,
                mean: 0,
                ts: null });
    }''')

`reducef`函数实现如下：

    reducef = bson.Code('''function(key, values) {
        var r = { total: 0, count: 0, mean: 0, ts: null };
        values.forEach(function(v) {
            r.total += v.total;
            r.count += v.count;
        });
        return r;
    }''')

`finalizef`函数实现如下：

    finalizef = bson.Code('''function(key, value) {
        if(value.count > 0) {
            value.mean = value.total / value.count;
        }
        value.ts = new Date();
        return value;
    }''')

`h_aggregate`函数实现如下：

    def h_aggregate(icollection, ocollection, mapf, cutoff, last_run):
        query = { 'value.ts': { '$gt': last_run, '$lt': cutoff } }
        icollection.map_reduce(
            map=mapf,
            reduce=reducef,
            finalize=finalizef,
            query=query,
            out={ 'reduce': ocollection.name, 'sharded': True })

    mapf_day = bson.Code(
        mapf_hierarchical % '''new Date(
            this._id.d.getFullYear(),
            this._id.d.getMonth(),
            this._id.d.getDate(),
            0, 0, 0, 0)''')

    mapf_hierarchical = '''function() {
        var key = {
            u: this._id.u,
            d: %s };
        emit(
            key,
            {
                total: this.value.total,
                count: this.value.count,
                mean: 0,
                ts: null });
    }'''

### Chapter 5 Ecommerce

#### 5.1 Product Catalog

MongoDB不是关系型数据库，所以可以将不同种类的商品存储在一张表里面。

**Index all the things**

In ecommerce systems, we typically don’t know exactly what the user will be filtering on, so it’s a good idea to create a number of indexes on queries that are likely to happen. Although such indexing will slow down updates, a product catalog is only very infrequently updated, so this drawback is justified by the significant improvements in search speed. To sum up, if your application has a code path to execute a query, there should be an index to accelerate that query.

**Sharding Concern**

这种查询密集型的应用，要尽量将数据根据常用查询，把同类数据局限到少数几个节点上。

**Scaling read performance without sharding**

索引尽量少，但是读数据时，尽量多。In these situations, you can gain some additional read performance by allowing mongos to read from the secondary mongod instances in a replica set by configuring the
read preference in the client.

#### 5.2 Category Hierarchy


