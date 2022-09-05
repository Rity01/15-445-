# Project 1 - BUFFER POOL

## 实验介绍
Buffer Pool和storage manager之间的关系。


建立一个面向磁盘的存储管理器。buffer pool负责在磁盘和内存之间搬运物理页。这样可以使得DBMS支持超过可用内存的数据库。buffer pool的操作对于其他部分透明。系统直接根据单独的标识（page_id_t）来问buffer pool内存，而不用管这个页面是否在内存上。


线程安全问题：Your implementation will need to be thread-safe. Multiple threads will be accessing the internal data structures at the same and thus you need to make sure that their critical sections are protected with latches 。

## TASK #1 - LRU REPLACEMENT POLICY
实现在src/include/buffer/lru_replacer.h 中的LRUReplacer类，这个类继承src/include/buffer/replacer.h中的Replacer类。

std::list<frame_id_t> frame_list_：LRUReplacer用来跟踪frame的列表。

Victim(frame_id_t*) :释放被LRUReplacer跟踪的frame（least recenttly）

Unpin(frame_id_t)：当page的pin_count变为0，调用此函数，把page对应的frame移入LRUReplacer（Victim可以释放此frame）。

Pin(frame_id_t)：当page被pin到frame中时调用，把page对应的frame移出LRUReplacer

## BUFFER POOL MANAGER

BufferPoolManagerInstance ：负责管理page在内存和磁盘中读取

Page：用来管理buffer pool中内存的对象。有自己的成员变量和方法。关键成员变量有data_  page_id_   pin_count_  is_dirty_  rwlatch_
	
	—page_id_:用来表示Page对象所保存的物理page页面。
	—pin_count_:用来表示此Page对象是否被pin，如果Pinned，则BufferPoolManage无法释放。（被移出LRU，并且也不在free_list_中）
				BufferPoolManagerInstance中FetchPgImp和NewPgImp会根据这两个list找到页面，然后读入覆盖。
	—is_dirty_：跟踪页面是否被修改。
				如果被修改，则在Victim之后，BufferPoolManagerInstance中FetchPgImp和NewPgImp读取新页面之前把dirty的页面写入磁盘。
				
BufferPoolManagerInstance：
	成员变量有pages_      free_list_  replacer_   page_table_  latch_  

	pages_：初始为缓冲池大小的Page数组。pages_ = new Page[pool_size_];
				pages_的大小和LRU的大小相同，pages_的数组索引即是frame_idx
	page_table_:映射page_id_t ----> frame_id_t (BufferPoolManagerInstance提供的接口是page_id的，方便用于找到对应的数组索引)
				FetchPgImp和NewPgImp中，如果需要移除Page对象的内容，要修改page_table_。
				
				
	对外接口：
	FetchPgImp：(1)增加pin_count_,并调用LRU的Pin（如果free list中没有或者所有的页都被pinned了，那么返回nullptr）
	NewPgImp：(1)不需要调用LRU的Pin
	UnpinPgImp：参数是page_id和is_dirty,减少通过page_id找到的Page对象的pin_count_，减少，如果变为0，则调用LRU的Unpin，如果新page没有空间，则可供释放。
	
	FlushPgImp：
	FlushAllPgsImp：
	DeletePgImp：
	
	
### 并发控制分析：哪些函数会同时访问page_中的同一对象？
pages_中Pin_count_：FetchPg和UnpinPg需要互斥，否则Fetch之后几乎同时Unpin，则Fetch的页面可能被释放。

FetchPg中找到page_中对象，并修改需要互斥访问（整体上，对page_对象进行修改pin_count，Pin（）的调用与NewPg寻找新的页面需要互斥，newPg对pages_修改page_id等也需在FetchPg之前或之后一并执行）
	
Page_table_:对page_table_作修改（移除，和添加），不应fetch修改的page_id
	
Free_list_: (1)std::list是否线程安全？需要互斥访问，不然同时调用NewPg或者FetchPage，可能给出相同的frame_idx。




## 源码分析

DiskManager负责数据库中页面的分配与deallocation，负责从从磁盘读取page。


构造函数
explicit DiskManager(const std::string &db_file);


在构造函数中打开文件的代码：
```C++
  std::scoped_lock scoped_db_io_latch(db_io_latch_);
  db_io_.open(db_file, std::ios::binary | std::ios::in | std::ios::out);
  // directory or file does not exist
  if (!db_io_.is_open()) {
    db_io_.clear();
    // create a new file
    db_io_.open(db_file, std::ios::binary | std::ios::trunc | std::ios::out);
    db_io_.close();
    // reopen with original mode
    db_io_.open(db_file, std::ios::binary | std::ios::in | std::ios::out);
    if (!db_io_.is_open()) {
      throw Exception("can't open db file");
    }
  }
  buffer_used = nullptr;
```


把文件写到磁盘文件（找到偏移后，调用write）

```C++
void DiskManager::WritePage(page_id_t page_id, const char *page_data) {
  std::scoped_lock scoped_db_io_latch(db_io_latch_);
  size_t offset = static_cast<size_t>(page_id) * PAGE_SIZE;
  // set write cursor to offset
  num_writes_ += 1;
  db_io_.seekp(offset);
  db_io_.write(page_data, PAGE_SIZE);
  // check for I/O error
  if (db_io_.bad()) {
    LOG_DEBUG("I/O error while writing");
    return;
  }
  // needs to flush to keep disk file in sync
  db_io_.flush();
}
```

写日志代码
```C++
/**
 * Write the contents of the log into disk file
 * Only return when sync is done, and only perform sequence write
 */
void DiskManager::WriteLog(char *log_data, int size) {
  // enforce swap log buffer
  assert(log_data != buffer_used);
  buffer_used = log_data;

  if (size == 0) {  // no effect on num_flushes_ if log buffer is empty
    return;
  }

  flush_log_ = true;

  if (flush_log_f_ != nullptr) {
    // used for checking non-blocking flushing
    assert(flush_log_f_->wait_for(std::chrono::seconds(10)) == std::future_status::ready);
  }

  num_flushes_ += 1;
  // sequence write
  log_io_.write(log_data, size);

  // check for I/O error
  if (log_io_.bad()) {
    LOG_DEBUG("I/O error while writing log");
    return;
  }
  // needs to flush to keep disk file in sync
  log_io_.flush();
  flush_log_ = false;
}
```


# Page对象分析
Page：用来管理buffer pool中内存的对象。有自己的成员变量和方法。成员变量有data_  page_id_   pin_count_  is_dirty_  rwlatch_。

But it is important for you as the system developer to understand that Page objects are just containers for memory in the buffer pool and thus are not specific to a unique page. That is, each Page object contains a block of memory that the DiskManager will use as a location to copy the contents of a physical page that it reads from disk. The BufferPoolManagerInstance will reuse the same Page object to store data as it moves back and forth to disk. This means that the same Page object may contain a different physical page throughout the life of the system. The Page object's identifer (page_id) keeps track of what physical page it contains; if a Page object does not contain a physical page, then its page_id must be set to INVALID_PAGE_ID.


Page对象中的data可以根据需要变动，比如TablePage就继承于Page类，在代码中通过转型来获取data中的信息。

实际的TablePage结构：

~~~
Slotted page format:
---------------------------------------------------------
| HEADER | ... FREE SPACE ... | ... INSERTED TUPLES ... |
---------------------------------------------------------
                              ^
                               free space pointer

Header format (size in bytes):
----------------------------------------------------------------------------
| PageId (4)| LSN (4)| PrevPageId (4)| NextPageId (4)| FreeSpacePointer(4) |
----------------------------------------------------------------------------
----------------------------------------------------------------
| TupleCount (4) | Tuple_1 offset (4) | Tuple_1 size (4) | ... |
----------------------------------------------------------------
 ~~~



比如解释并访问Page中信息代码如下：
```C++
  /** @return the page ID of this table page */
  page_id_t GetTablePageId() { return *reinterpret_cast<page_id_t *>(GetData()); }

  /** @return the page ID of the previous table page */
  page_id_t GetPrevPageId() { return *reinterpret_cast<page_id_t *>(GetData() + OFFSET_PREV_PAGE_ID); }
```

此外TablePage提供了InsertTuple和UpdateTuple、GetTuple等功能。
