# CONCURRENCY CONTROL
实现一个lock manager来支持并发query的执行。lock manager负责跟踪tuple级的锁，根据不同的隔离等级来分发锁。


## TASK #1 - LOCK MANAGER

lock manager会维护一个内部的数据结构（现在被活跃的事务保持的锁），在事务被允许访问数据前事务会向LM提出锁申请，LM会grant锁、block事务或者abort事务。

The TableHeap and Executor classes will use your LM to acquire locks on tuple records (by record id RID) whenever a transaction wants to access/modify a tuple.

实现一个tuple—level的LM来支持三种隔离等级：READ_UNCOMMITED, READ_COMMITTED, 和REPEATABLE_READ. 


Transaction内部记录了tuple和index的操作数据，这样tansaction_manager类能对这个tansaction的操作进行回滚。还维护了在tuple中获得的锁，这样commit或者abort的时候可以释放所有的锁。内部成员变量如下
```C++
  /** The undo set of table tuples. */
  std::shared_ptr<std::deque<TableWriteRecord>> table_write_set_;
  /** The undo set of indexes. */
  std::shared_ptr<std::deque<IndexWriteRecord>> index_write_set_;
  /** The LSN of the last record written by the transaction. */
  lsn_t prev_lsn_;

  /** Concurrent index: the pages that were latched during index operation. */
  std::shared_ptr<std::deque<Page *>> page_set_;
  /** Concurrent index: the page IDs that were deleted during index operation.*/
  std::shared_ptr<std::unordered_set<page_id_t>> deleted_page_set_;

  /** LockManager: the set of shared-locked tuples held by this transaction. */
  std::shared_ptr<std::unordered_set<RID>> shared_lock_set_;
  /** LockManager: the set of exclusive-locked tuples held by this transaction. */
  std::shared_ptr<std::unordered_set<RID>> exclusive_lock_set_;
```

TansactionManager类的Abort函数如下：
```C++
void TransactionManager::Abort(Transaction *txn) {
  txn->SetState(TransactionState::ABORTED);
  // Rollback before releasing the lock.
  auto table_write_set = txn->GetWriteSet();
  while (!table_write_set->empty()) {
    auto &item = table_write_set->back();
    auto table = item.table_;
    if (item.wtype_ == WType::DELETE) {
      table->RollbackDelete(item.rid_, txn);
    } else if (item.wtype_ == WType::INSERT) {
      // Note that this also releases the lock when holding the page latch.
      table->ApplyDelete(item.rid_, txn);
    } else if (item.wtype_ == WType::UPDATE) {
      table->UpdateTuple(item.tuple_, item.rid_, txn);
    }
    table_write_set->pop_back();
  }
  table_write_set->clear();
  // Rollback index updates
  auto index_write_set = txn->GetIndexWriteSet();
  while (!index_write_set->empty()) {
    auto &item = index_write_set->back();
    auto catalog = item.catalog_;
    // Metadata identifying the table that should be deleted from.
    TableInfo *table_info = catalog->GetTable(item.table_oid_);
    IndexInfo *index_info = catalog->GetIndex(item.index_oid_);
    auto new_key = item.tuple_.KeyFromTuple(table_info->schema_, *(index_info->index_->GetKeySchema()),
                                            index_info->index_->GetKeyAttrs());
    if (item.wtype_ == WType::DELETE) {
      index_info->index_->InsertEntry(new_key, item.rid_, txn);
    } else if (item.wtype_ == WType::INSERT) {
      index_info->index_->DeleteEntry(new_key, item.rid_, txn);
    } else if (item.wtype_ == WType::UPDATE) {
      // Delete the new key and insert the old key
      index_info->index_->DeleteEntry(new_key, item.rid_, txn);
      auto old_key = item.old_tuple_.KeyFromTuple(table_info->schema_, *(index_info->index_->GetKeySchema()),
                                                  index_info->index_->GetKeyAttrs());
      index_info->index_->InsertEntry(old_key, item.rid_, txn);
    }
    index_write_set->pop_back();
  }
  table_write_set->clear();
  index_write_set->clear();

  // Release all the locks.
  ReleaseLocks(txn);
  // Release the global transaction latch.
  global_txn_latch_.RUnlock();
}
```

LockManager内部有std::unordered_map<RID, LockRequestQueue> lock_table_这样的锁表结构，会提供事务申请S锁，X锁，和更新锁的成员函数，这些函数事务当前的状态，表锁内容，隔离等级决定是否分配锁，如果出现违规申请锁，那么会设置事务状态为TransactionState::ABORTED，直接抛出异常回滚事务。如果暂时不满足要求，那么事务会等待在条件变量上面，知道其他事务因为释放锁来通知改事务，那么这个事务会重新判断自己能否获得锁，从而判断是否应该继续阻塞或者获得锁。

```C++
bool LockManager::LockShared(Transaction *txn, const RID &rid) {
  std::unique_lock lk(lock_);
  if (txn->GetIsolationLevel() == IsolationLevel::READ_UNCOMMITTED) {
    // RU隔离等级直接回滚
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::LOCK_ON_SHRINKING);
    return false;
  }
  if (txn->GetState() == TransactionState::SHRINKING) {
    // 若违反两相锁协议则直接设置为ABORTED,抛出异常
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(), AbortReason::LOCK_ON_SHRINKING);
    return false;
  }

  // 在插入之前，进行would-wait检测
  for (auto &request : lock_table_[rid].request_queue_) {
    if (txn->GetTransactionId() < request.txn_id_ && request.lock_mode_ == LockMode::EXCLUSIVE) {
      TransactionManager::GetTransaction(request.txn_id_)->SetState(TransactionState::ABORTED);

      lock_table_[rid].cv_.notify_all();
    }
  }

  lock_table_[rid].request_queue_.emplace_back(LockRequest{txn->GetTransactionId(), LockMode::SHARED});

  lock_table_[rid].cv_.wait(lk, [this, txn, rid] { return this->GetSharedLock(txn, rid); });

  txn->GetSharedLockSet()->emplace(rid);

  return true;
}
```
## 死锁预防

使用would-wait算法可以提供锁预防的功能，这个算法思想很简单，如果事务相比先持有锁的事务更老（id更小），那么直接把那些事务abort并释放锁。

## Query Execution的修改
上面是讲到的LockManager是负责是不是应该grant锁，但是什么时候申请锁（对于RU甚至不用申请锁）其实还是应该由事务自身决定的，所以需要对执行器做修改，对于释放锁的操作，比如说是RC的隔离等级，那么S锁会立即释放。比如对于insert执行器的申请锁如下所示，（可以看到rid是先插入生成在加锁）
```C++
if (child_executor_->Next(tuple, rid)) {
  res = (table_info_->table_)->InsertTuple(*tuple, rid, exec_ctx_->GetTransaction());
  //插入并法控制
  Transaction *txn = exec_ctx_->GetTransaction();
  LockManager *lock_mgr = exec_ctx_->GetLockManager();
  if (lock_mgr != nullptr) {
    if (!txn->IsExclusiveLocked(*rid)) {
      if (!txn->IsSharedLocked(*rid)) {
        lock_mgr->LockExclusive(txn, *rid);
      } else {
        lock_mgr->LockUpgrade(txn, *rid);
      }
    }
  }
  if (res && !index_info_.empty()) {
    for (auto iter : index_info_) {
      (iter->index_)
          ->InsertEntry(tuple->KeyFromTuple(table_info_->schema_, iter->key_schema_, (iter->index_)->GetKeyAttrs()),
                        *rid, exec_ctx_->GetTransaction());
      txn->AppendTableWriteRecord(IndexWriteRecord(*rid, table_info_->oid_, WType::INSERT, *tuple, iter->index_oid_,
                                                    exec_ctx_->GetCatalog()));
    }
  }
```