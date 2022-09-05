# 实现一个disk—backed的哈希表

## HASH TABLE DIRECTORY PAGE

存储hash表的mata数据

    bucket_page_ids_：把bucket的id映射为page_id_t。第i个元素即为bucket的page_id

    page_id_，lsn_
    global_depth_和local_depth_


HashTableDirectoryPage并不继承Page对象，所以直接把页面的内容通过reinterpret_cast转换类型得到。`auto directory_page =reinterpret_cast<HashTableDirectoryPage *>(bpm->NewPage(&directory_page_id, nullptr)->GetData());`

global_depth_用于计算索引，可以得到GLOBAL_DEPTH_MASK，DirectoryIndex = Hash(key) & GLOBAL_DEPTH_MASK，比如说3对应于0x00000007

增加global_depth时，应该让新的local_depth和bucket_page_ids_指向和原来对应的东西，代码如下：
```C++
void HashTableDirectoryPage::IncrGlobalDepth() {
  uint32_t size = 1 << global_depth_;
  global_depth_++;
  if (global_depth_ <= 1) {
    return;
  }
  memcpy(&local_depths_[size], local_depths_, size * sizeof(uint8_t));
  memcpy(&bucket_page_ids_[size], bucket_page_ids_, size * sizeof(page_id_t));
}
```

local_depths是一个数组，`uint8_t local_depths_[DIRECTORY_ARRAY_SIZE];`
GetLocalDepthMask根据bucket_idx返回local的掩码。 

`page_id_t HashTableDirectoryPage::GetBucketPageId(uint32_t bucket_idx)`根据bucket_idx找到page_id



## HASH TABLE BUCKET PAGE
    occupied_
    readable_
    array_  保存key value对的array，代码只需支持固定长度的键值对

hash table bucket page支持 Insert - Remove - IsOccupied - IsReadable - KeyAt - ValueAt等功能。


# HASH TABLE IMPLEMENTATION

使用模板定义ExtendibleHashTable
`template <typename KeyType, typename ValueType, typename KeyComparator>
class ExtendibleHashTable{}` 

有一个变量directory_page_id_指向DIRECTORY PAGE

bucket已满，当插入时调用SplitInsert，代码如下：

```C++
template <typename KeyType, typename ValueType, typename KeyComparator>
bool HASH_TABLE_TYPE::SplitInsert(Transaction *transaction, const KeyType &key, const ValueType &value) {
  auto dir_page = FetchDirectoryPage();
  while (dir_page == nullptr) {
    dir_page = FetchDirectoryPage();
  }
  LOG_INFO("begin split insert");
  dir_page->PrintDirectory();
  uint32_t idx = KeyToDirectoryIndex(key, dir_page);
  page_id_t bucket_page_id = KeyToPageId(key, dir_page);
  auto bkt_page = FetchBucketPage(bucket_page_id);
  while (bkt_page == nullptr) {
    bkt_page = FetchBucketPage(bucket_page_id);
    LOG_INFO("try to fetch bkt_page to insert key_value");
  }
  while (bkt_page->IsFull()) {
    if (dir_page->GetGlobalDepth() == dir_page->GetLocalDepth(idx)) {
      // 应该判断是否失败
      dir_page->IncrGlobalDepth();
    }
    page_id_t bucket_page_id_new = INVALID_PAGE_ID;
    Page *page_object_bucket_new = buffer_pool_manager_->NewPage(&bucket_page_id_new);
    auto bkt_page_new =
        reinterpret_cast<HashTableBucketPage<KeyType, ValueType, KeyComparator> *>(page_object_bucket_new->GetData());

    for (uint32_t i = 0; i < BUCKET_ARRAY_SIZE; i++) {
      auto k = bkt_page->KeyAt(i);
      auto v = bkt_page->ValueAt(i);
      uint32_t depth = dir_page->GetLocalDepth(idx);
      uint32_t m = (1 << (depth + 1)) - 1;
      if ((Hash(k) & m) != (idx & m)) {
        bkt_page_new->Insert(k, v, comparator_);
        bkt_page->RemoveAt(i);
      }
    }
    dir_page->IncrLocalDepth(idx);
    uint32_t idx_recursive;
    for (uint32_t j = 0; j < dir_page->Size(); j++) {
      uint32_t mask = (1 << (dir_page->GetLocalDepth(idx) - 1)) - 1;
      uint32_t high_bit = 1 << (dir_page->GetLocalDepth(idx) - 1);
      if ((j & mask) - (idx & mask) == 0) {
        if ((j & high_bit) == (idx & high_bit)) {
          dir_page->SetLocalDepth(j, dir_page->GetLocalDepth(idx));
        } else {
          dir_page->SetLocalDepth(j, dir_page->GetLocalDepth(idx));
          dir_page->SetBucketPageId(j, bucket_page_id_new);
          idx_recursive = j;
        }
      }
    }
    if (bkt_page->IsFull()) {
      buffer_pool_manager_->UnpinPage(bucket_page_id_new, true);
    } else if (bkt_page_new->IsFull()) {
      bkt_page = bkt_page_new;
      idx = idx_recursive;
      bucket_page_id = bucket_page_id_new;
      buffer_pool_manager_->UnpinPage(bucket_page_id, true);
    } else {
      uint32_t idx_to_insert = KeyToDirectoryIndex(key, dir_page);
      bool result;
      if (idx_to_insert == idx) {
        result = bkt_page->Insert(key, value, comparator_);
      } else {
        result = bkt_page_new->Insert(key, value, comparator_);
      }

      buffer_pool_manager_->UnpinPage(bucket_page_id_new, true);
      buffer_pool_manager_->UnpinPage(bucket_page_id, true);
      LOG_INFO("after split insert");
      dir_page->PrintDirectory();
      return result;
    }
  }
  // 不会执行下面的语句
  return false;
}
```
