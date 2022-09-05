# Query Execution

实现excutors，来执行query plan nodes。使用火山模型来处理query，每一个excutor实现一个Next函数，当DMBS调用一个excutor的Next函数，excutor返回一个tuple或者indicator表明没有tuple了。使用这种方法，每一个excutor实现一个循环不断调用它的子节点来取出tuple并一个一个处理他们。

每个excutor负责处理一个plan node类型，Plan nodes are the individual elements that compose a query plan.

plan nodes已经拥有实现需要的excutors所需的数据和函数，excutor从他们的孩子接收tuple，并产生tuple到他们的父亲。

ExcutionEngine会把输入的query plan转换为query executor，并且执行它直到所有结果被产出。You must modify the ExecutionEngine to catch any exception that your executors throw as a result of failures during query execution. Proper handling of failures in the execution engine will be important for success on future projects.

```C++
  /**
   * Execute a query plan.
   * @param plan The query plan to execute
   * @param result_set The set of tuples produced by executing the plan
   * @param txn The transaction context in which the query executes
   * @param exec_ctx The executor context in which the query executes
   * @return `true` if execution of the query plan succeeds, `false` otherwise
   */
  bool Execute(const AbstractPlanNode *plan, std::vector<Tuple> *result_set, Transaction *txn,
               ExecutorContext *exec_ctx) {
    // Construct and executor for the plan
    auto executor = ExecutorFactory::CreateExecutor(exec_ctx, plan);

    // Prepare the root executor
    executor->Init();

    // Execute the query plan
    try {
      Tuple tuple;
      RID rid;
      while (executor->Next(&tuple, &rid)) {
        if (result_set != nullptr) {
          result_set->push_back(tuple);
        }
      }
    } catch (Exception &e) {
      // TODO(student): handle exceptions
    }

    return true;
  }
```


ExcutorFactor会根据传入AbstractPlanNode类型的type调用不同的构造函数，代码如下：
```C++
std::unique_ptr<AbstractExecutor> ExecutorFactory::CreateExecutor(ExecutorContext *exec_ctx,
                                                                  const AbstractPlanNode *plan) {
  switch (plan->GetType()) {
    // Create a new sequential scan executor
    case PlanType::SeqScan: {
      return std::make_unique<SeqScanExecutor>(exec_ctx, dynamic_cast<const SeqScanPlanNode *>(plan));
    }

    // Create a new index scan executor
    case PlanType::IndexScan: {
      return std::make_unique<IndexScanExecutor>(exec_ctx, dynamic_cast<const IndexScanPlanNode *>(plan));
    }
  }
```

ExcutorContext包含query执行的上下文信息，代码如下：
```C++
class ExecutorContext {
 public:
  /**
   * Creates an ExecutorContext for the transaction that is executing the query.
   * @param transaction The transaction executing the query
   * @param catalog The catalog that the executor uses
   * @param bpm The buffer pool manager that the executor uses
   * @param txn_mgr The transaction manager that the executor uses
   * @param lock_mgr The lock manager that the executor uses
   */
  ExecutorContext(Transaction *transaction, Catalog *catalog, BufferPoolManager *bpm, TransactionManager *txn_mgr,
                  LockManager *lock_mgr)
      : transaction_(transaction), catalog_{catalog}, bpm_{bpm}, txn_mgr_(txn_mgr), lock_mgr_(lock_mgr) {}
}
```


可以从catalog中返回TableInfo，TableInfo的结构如下：
```C++
/**
 * The TableInfo class maintains metadata about a table.
 */
struct TableInfo {
  /**
   * Construct a new TableInfo instance.
   * @param schema The table schema
   * @param name The table name
   * @param table An owning pointer to the table heap
   * @param oid The unique OID for the table
   */
  TableInfo(Schema schema, std::string name, std::unique_ptr<TableHeap> &&table, table_oid_t oid)
      : schema_{std::move(schema)}, name_{std::move(name)}, table_{std::move(table)}, oid_{oid} {}
  /** The table schema */
  Schema schema_;
  /** The table name */
  const std::string name_;
  /** An owning pointer to the table heap */
  std::unique_ptr<TableHeap> table_;
  /** The table OID */
  const table_oid_t oid_;
};
```

TableHeap完成了对table的操作，并且提供了遍历table的迭代器，代码如下：
```C++
//===----------------------------------------------------------------------===//
//
//                         BusTub
//
// table_heap.cpp
//
// Identification: src/storage/table/table_heap.cpp
//
// Copyright (c) 2015-2019, Carnegie Mellon University Database Group
//
//===----------------------------------------------------------------------===//

#include <cassert>

#include "common/logger.h"
#include "storage/table/table_heap.h"

namespace bustub {

TableHeap::TableHeap(BufferPoolManager *buffer_pool_manager, LockManager *lock_manager, LogManager *log_manager,
                     page_id_t first_page_id)
    : buffer_pool_manager_(buffer_pool_manager),
      lock_manager_(lock_manager),
      log_manager_(log_manager),
      first_page_id_(first_page_id) {}

TableHeap::TableHeap(BufferPoolManager *buffer_pool_manager, LockManager *lock_manager, LogManager *log_manager,
                     Transaction *txn)
    : buffer_pool_manager_(buffer_pool_manager), lock_manager_(lock_manager), log_manager_(log_manager) {
  // Initialize the first table page.
  auto first_page = reinterpret_cast<TablePage *>(buffer_pool_manager_->NewPage(&first_page_id_));
  BUSTUB_ASSERT(first_page != nullptr, "Couldn't create a page for the table heap.");
  first_page->WLatch();
  first_page->Init(first_page_id_, PAGE_SIZE, INVALID_LSN, log_manager_, txn);
  first_page->WUnlatch();
  buffer_pool_manager_->UnpinPage(first_page_id_, true);
}

bool TableHeap::InsertTuple(const Tuple &tuple, RID *rid, Transaction *txn) {
  if (tuple.size_ + 32 > PAGE_SIZE) {  // larger than one page size
    txn->SetState(TransactionState::ABORTED);
    return false;
  }

  auto cur_page = static_cast<TablePage *>(buffer_pool_manager_->FetchPage(first_page_id_));
  if (cur_page == nullptr) {
    txn->SetState(TransactionState::ABORTED);
    return false;
  }

  cur_page->WLatch();
  // Insert into the first page with enough space. If no such page exists, create a new page and insert into that.
  // INVARIANT: cur_page is WLatched if you leave the loop normally.
  while (!cur_page->InsertTuple(tuple, rid, txn, lock_manager_, log_manager_)) {
    auto next_page_id = cur_page->GetNextPageId();
    // If the next page is a valid page,
    if (next_page_id != INVALID_PAGE_ID) {
      // Unlatch and unpin the current page.
      cur_page->WUnlatch();
      buffer_pool_manager_->UnpinPage(cur_page->GetTablePageId(), false);
      // And repeat the process with the next page.
      cur_page = static_cast<TablePage *>(buffer_pool_manager_->FetchPage(next_page_id));
      cur_page->WLatch();
    } else {
      // Otherwise we have run out of valid pages. We need to create a new page.
      auto new_page = static_cast<TablePage *>(buffer_pool_manager_->NewPage(&next_page_id));
      // If we could not create a new page,
      if (new_page == nullptr) {
        // Then life sucks and we abort the transaction.
        cur_page->WUnlatch();
        buffer_pool_manager_->UnpinPage(cur_page->GetTablePageId(), false);
        txn->SetState(TransactionState::ABORTED);
        return false;
      }
      // Otherwise we were able to create a new page. We initialize it now.
      new_page->WLatch();
      cur_page->SetNextPageId(next_page_id);
      new_page->Init(next_page_id, PAGE_SIZE, cur_page->GetTablePageId(), log_manager_, txn);
      cur_page->WUnlatch();
      buffer_pool_manager_->UnpinPage(cur_page->GetTablePageId(), true);
      cur_page = new_page;
    }
  }
  // This line has caused most of us to double-take and "whoa double unlatch".
  // We are not, in fact, double unlatching. See the invariant above.
  cur_page->WUnlatch();
  buffer_pool_manager_->UnpinPage(cur_page->GetTablePageId(), true);
  // Update the transaction's write set.
  txn->GetWriteSet()->emplace_back(*rid, WType::INSERT, Tuple{}, this);
  return true;
}

bool TableHeap::MarkDelete(const RID &rid, Transaction *txn) {
  // TODO(Amadou): remove empty page
  // Find the page which contains the tuple.
  auto page = reinterpret_cast<TablePage *>(buffer_pool_manager_->FetchPage(rid.GetPageId()));
  // If the page could not be found, then abort the transaction.
  if (page == nullptr) {
    txn->SetState(TransactionState::ABORTED);
    return false;
  }
  // Otherwise, mark the tuple as deleted.
  page->WLatch();
  page->MarkDelete(rid, txn, lock_manager_, log_manager_);
  page->WUnlatch();
  buffer_pool_manager_->UnpinPage(page->GetTablePageId(), true);
  // Update the transaction's write set.
  txn->GetWriteSet()->emplace_back(rid, WType::DELETE, Tuple{}, this);
  return true;
}

bool TableHeap::UpdateTuple(const Tuple &tuple, const RID &rid, Transaction *txn) {
  // Find the page which contains the tuple.
  auto page = reinterpret_cast<TablePage *>(buffer_pool_manager_->FetchPage(rid.GetPageId()));
  // If the page could not be found, then abort the transaction.
  if (page == nullptr) {
    txn->SetState(TransactionState::ABORTED);
    return false;
  }
  // Update the tuple; but first save the old value for rollbacks.
  Tuple old_tuple;
  page->WLatch();
  bool is_updated = page->UpdateTuple(tuple, &old_tuple, rid, txn, lock_manager_, log_manager_);
  page->WUnlatch();
  buffer_pool_manager_->UnpinPage(page->GetTablePageId(), is_updated);
  // Update the transaction's write set.
  if (is_updated && txn->GetState() != TransactionState::ABORTED) {
    txn->GetWriteSet()->emplace_back(rid, WType::UPDATE, old_tuple, this);
  }
  return is_updated;
}

void TableHeap::ApplyDelete(const RID &rid, Transaction *txn) {
  // Find the page which contains the tuple.
  auto page = reinterpret_cast<TablePage *>(buffer_pool_manager_->FetchPage(rid.GetPageId()));
  BUSTUB_ASSERT(page != nullptr, "Couldn't find a page containing that RID.");
  // Delete the tuple from the page.
  page->WLatch();
  page->ApplyDelete(rid, txn, log_manager_);
  lock_manager_->Unlock(txn, rid);
  page->WUnlatch();
  buffer_pool_manager_->UnpinPage(page->GetTablePageId(), true);
}

void TableHeap::RollbackDelete(const RID &rid, Transaction *txn) {
  // Find the page which contains the tuple.
  auto page = reinterpret_cast<TablePage *>(buffer_pool_manager_->FetchPage(rid.GetPageId()));
  BUSTUB_ASSERT(page != nullptr, "Couldn't find a page containing that RID.");
  // Rollback the delete.
  page->WLatch();
  page->RollbackDelete(rid, txn, log_manager_);
  page->WUnlatch();
  buffer_pool_manager_->UnpinPage(page->GetTablePageId(), true);
}

bool TableHeap::GetTuple(const RID &rid, Tuple *tuple, Transaction *txn) {
  // Find the page which contains the tuple.
  auto page = static_cast<TablePage *>(buffer_pool_manager_->FetchPage(rid.GetPageId()));
  // If the page could not be found, then abort the transaction.
  if (page == nullptr) {
    txn->SetState(TransactionState::ABORTED);
    return false;
  }
  // Read the tuple from the page.
  page->RLatch();
  bool res = page->GetTuple(rid, tuple, txn, lock_manager_);
  page->RUnlatch();
  buffer_pool_manager_->UnpinPage(rid.GetPageId(), false);
  return res;
}

TableIterator TableHeap::Begin(Transaction *txn) {
  // Start an iterator from the first page.
  // TODO(Wuwen): Hacky fix for now. Removing empty pages is a better way to handle this.
  RID rid;
  auto page_id = first_page_id_;
  while (page_id != INVALID_PAGE_ID) {
    auto page = static_cast<TablePage *>(buffer_pool_manager_->FetchPage(page_id));
    page->RLatch();
    // If this fails because there is no tuple, then RID will be the default-constructed value, which means EOF.
    auto found_tuple = page->GetFirstTupleRid(&rid);
    page->RUnlatch();
    buffer_pool_manager_->UnpinPage(page_id, false);
    if (found_tuple) {
      break;
    }
    page_id = page->GetNextPageId();
  }
  return TableIterator(this, rid, txn);
}

TableIterator TableHeap::End() { return TableIterator(this, RID(INVALID_PAGE_ID, 0), nullptr); }

}  // namespace bustub
```

迭代器会根据RID来重载++运算符，代码如下：

```C++
class RID {
 public:
  /** The default constructor creates an invalid RID! */
  RID() = default;

  /**
   * Creates a new Record Identifier for the given page identifier and slot number.
   * @param page_id page identifier
   * @param slot_num slot number
   */
  RID(page_id_t page_id, uint32_t slot_num) : page_id_(page_id), slot_num_(slot_num) {}

  explicit RID(int64_t rid) : page_id_(static_cast<page_id_t>(rid >> 32)), slot_num_(static_cast<uint32_t>(rid)) {}

  inline int64_t Get() const { return (static_cast<int64_t>(page_id_)) << 32 | slot_num_; }

  inline page_id_t GetPageId() const { return page_id_; }

  inline uint32_t GetSlotNum() const { return slot_num_; }

  inline void Set(page_id_t page_id, uint32_t slot_num) {
    page_id_ = page_id;
    slot_num_ = slot_num;
  }

  inline std::string ToString() const {
    std::stringstream os;
    os << "page_id: " << page_id_;
    os << " slot_num: " << slot_num_ << "\n";

    return os.str();
  }

  friend std::ostream &operator<<(std::ostream &os, const RID &rid) {
    os << rid.ToString();
    return os;
  }

  bool operator==(const RID &other) const { return page_id_ == other.page_id_ && slot_num_ == other.slot_num_; }

 private:
  page_id_t page_id_{INVALID_PAGE_ID};
  uint32_t slot_num_{0};  // logical offset from 0, 1...
};

}  // namespace bustub
```

* SeqScanExecutor代码如下：
```C++
namespace bustub {

// 或者使用iterator类，并初始化
SeqScanExecutor::SeqScanExecutor(ExecutorContext *exec_ctx, const SeqScanPlanNode *plan) : AbstractExecutor(exec_ctx),iter_now_(TableIterator{nullptr, RID{}, nullptr}) {
  plan_ = plan;
  // iter_now_ = new TableIterator(nullptr, RID{}, nullptr);
}

void SeqScanExecutor::Init() {
  std::cout << "Init start" << std::endl;
  table_oid_t table_oid = plan_->GetTableOid();
  predicate_ = plan_->GetPredicate();
  Catalog *catalog = exec_ctx_->GetCatalog();
  table_info_ = catalog->GetTable(table_oid);
  assert(table_info_ != Catalog::NULL_TABLE_INFO);
  // p_table_heap_ = table_info_->table_;
  (iter_now_) = (table_info_->table_)->Begin(exec_ctx_->GetTransaction());
  std::cout << "Init over" << std::endl;
}

bool SeqScanExecutor::Next(Tuple *tuple, RID *rid) {
  while (iter_now_ != (table_info_->table_)->End()) {
    // std::cout << "scan" << std::endl;
    const Tuple tuple_temp = *iter_now_;
    iter_now_++;
    if(predicate_ == nullptr){
      *tuple = tuple_temp;
      *rid = tuple_temp.GetRid();
      return true;
    }
    auto res = predicate_->Evaluate(&tuple_temp, &(table_info_->schema_));
    if (res.GetAs<bool>()) {
      *tuple = tuple_temp;
      *rid = tuple_temp.GetRid();
      return true;
    }
  }
  return false;
}
}  // namespace bustub
```

* UpdateExecutor会根据child_executor_返回的tuple来建立新的tuple，并根据rid把新的tuple更新到对应位置上。
```C++
    Tuple tuple_new = GenerateUpdatedTuple(*tuple);
    bool res = (table_info_->table_)->UpdateTuple(tuple_new,*rid,exec_ctx_->GetTransaction());
```

* 对于嵌套Join，需要左右两个子查询器，保证左边迭代器不变，直到右边的结束在重新初始化左边的子查询器。其构造函数如下：
```C++
NestedLoopJoinExecutor::NestedLoopJoinExecutor(ExecutorContext *exec_ctx, const NestedLoopJoinPlanNode *plan,
                                               std::unique_ptr<AbstractExecutor> &&left_executor,
                                               std::unique_ptr<AbstractExecutor> &&right_executor)
    : AbstractExecutor(exec_ctx),
      plan_(plan),
      left_executor_(std::move(left_executor)),
      right_executor_(std::move(right_executor)) {
  predicate_ = plan_->Predicate();
  output_schema_ = plan_->OutputSchema();
}
```
* 对于hash join,需要把左边所有的tuple建立一个哈希表，实验中只要求建立容纳在内存中的哈希表，实际情况时哈希表有可能在内存中放不下，hash join是一个 pipeline breaker，因为要建立哈希表必须等待所有的左子查询结束才能建立完整的哈希表。

* 对于聚合操作，其中包含了group by，聚合，和having语句，执行计划类如下所示：
```C++
 AggregationPlanNode(const Schema *output_schema, const AbstractPlanNode *child, const AbstractExpression *having,
                      std::vector<const AbstractExpression *> &&group_bys,
                      std::vector<const AbstractExpression *> &&aggregates, std::vector<AggregationType> &&agg_types)
      : AbstractPlanNode(output_schema, {child}),
        having_(having),
        group_bys_(std::move(group_bys)),
        aggregates_(std::move(aggregates)),
        agg_types_(std::move(agg_types)) {}

  /** @return The type of the plan node */
  PlanType GetType() const override { return PlanType::Aggregation; }

  /** @return the child of this aggregation plan node */
  const AbstractPlanNode *GetChildPlan() const {
    BUSTUB_ASSERT(GetChildren().size() == 1, "Aggregation expected to only have one child.");
    return GetChildAt(0);
  }
``` 
聚合操作也会建立一个哈希表SimpleAggregationHashTable，成员变量如下：
```C++
/** The hash table is just a map from aggregate keys to aggregate values */
  std::unordered_map<AggregateKey, AggregateValue> ht_{};
  /** The aggregate expressions that we have */
  const std::vector<const AbstractExpression *> &agg_exprs_;
  /** The types of aggregations that we have */
  const std::vector<AggregationType> &agg_types_;
```
其中哈希表中的key是AggregateKey，这个key是由plan_->GetGroupBys()得到，在插入的时候也会随时更新count数，总数，最大值，最小值
```C++
void InsertCombine(const AggregateKey &agg_key, const AggregateValue &agg_val) {
    if (ht_.count(agg_key) == 0) {
      ht_.insert({agg_key, GenerateInitialAggregateValue()});
    }
    CombineAggregateValues(&ht_[agg_key], agg_val);
  }
```

聚合操作执行器next函数如下：
```C++
bool AggregationExecutor::Next(Tuple *tuple, RID *rid) {
  while (aht_iterator_ != aht_.End()) {
    auto agg_key = aht_iterator_.Key();
    auto agg_value = aht_iterator_.Val();
    if (plan_->GetHaving() == nullptr ||
        plan_->GetHaving()->EvaluateAggregate(agg_key.group_bys_, agg_value.aggregates_).GetAs<bool>()) {
      std::vector<Value> ret;
      for (const auto &col : plan_->OutputSchema()->GetColumns()) {
        ret.push_back(col.GetExpr()->EvaluateAggregate(agg_key.group_bys_, agg_value.aggregates_));
      }
      *tuple = Tuple(ret, plan_->OutputSchema());
      ++aht_iterator_;
      return true;
    }
    ++aht_iterator_;
  }
  return false;
}
```

支持这样的**SELECT count(col_a), col_b, sum(col_c) FROM test_1 Group By col_b HAVING count(col_a) > 100** ，构造的测试代码如下：
```C++
// SELECT count(col_a), col_b, sum(col_c) FROM test_1 Group By col_b HAVING count(col_a) > 100
TEST_F(ExecutorTest, SimpleGroupByAggregation) {
  const Schema *scan_schema;
  std::unique_ptr<AbstractPlanNode> scan_plan;
  {
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    auto col_a = MakeColumnValueExpression(schema, 0, "colA");
    auto col_b = MakeColumnValueExpression(schema, 0, "colB");
    auto col_c = MakeColumnValueExpression(schema, 0, "colC");
    scan_schema = MakeOutputSchema({{"colA", col_a}, {"colB", col_b}, {"colC", col_c}});
    scan_plan = std::make_unique<SeqScanPlanNode>(scan_schema, nullptr, table_info->oid_);
  }

  const Schema *agg_schema;
  std::unique_ptr<AbstractPlanNode> agg_plan;
  {
    const AbstractExpression *col_a = MakeColumnValueExpression(*scan_schema, 0, "colA");
    const AbstractExpression *col_b = MakeColumnValueExpression(*scan_schema, 0, "colB");
    const AbstractExpression *col_c = MakeColumnValueExpression(*scan_schema, 0, "colC");
    // Make group bys
    std::vector<const AbstractExpression *> group_by_cols{col_b};
    const AbstractExpression *groupby_b = MakeAggregateValueExpression(true, 0);
    // Make aggregates
    std::vector<const AbstractExpression *> aggregate_cols{col_a, col_c};
    std::vector<AggregationType> agg_types{AggregationType::CountAggregate, AggregationType::SumAggregate};
    const AbstractExpression *count_a = MakeAggregateValueExpression(false, 0);
    // Make having clause
    const AbstractExpression *having = MakeComparisonExpression(
        count_a, MakeConstantValueExpression(ValueFactory::GetIntegerValue(100)), ComparisonType::GreaterThan);

    // Create plan
    agg_schema = MakeOutputSchema({{"countA", count_a}, {"colB", groupby_b}});
    agg_plan = std::make_unique<AggregationPlanNode>(agg_schema, scan_plan.get(), having, std::move(group_by_cols),
                                                     std::move(aggregate_cols), std::move(agg_types));
  }

  std::vector<Tuple> result_set{};
  GetExecutionEngine()->Execute(agg_plan.get(), &result_set, GetTxn(), GetExecutorContext());

  std::unordered_set<int32_t> encountered{};
  for (const auto &tuple : result_set) {
    // Should have count_a > 100
    ASSERT_GT(tuple.GetValue(agg_schema, agg_schema->GetColIdx("countA")).GetAs<int32_t>(), 100);
    // Should have unique col_bs.
    auto col_b = tuple.GetValue(agg_schema, agg_schema->GetColIdx("colB")).GetAs<int32_t>();
    ASSERT_EQ(encountered.count(col_b), 0);
    encountered.insert(col_b);
    // Sanity check: col_b should also be within [0, 10).
    ASSERT_TRUE(0 <= col_b && col_b < 10);
  }
}
```