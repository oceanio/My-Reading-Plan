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

使用MongoDB存储日志



