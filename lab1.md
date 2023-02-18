# Lab1

实验基于2020版本的bustub。

lab1要实现一个buffer pool，里面包含了一个基于LRU replacer以及各种操作page的方法。在开始之前，首先要了解一下frame、page、page_table这三者的关系。

在Buffer Pool中，内存被组织为一个个大小与page相同的frame(页帧)。这些frame被一个叫free_list的链表组织起来，而page相当于一个个的容器，这个容器里放的就是从磁盘中读取的内容，page会通过page_table一个个绑定到frame上。在buffer pool中，`page_table`的映射方向为page_id->frame_id，而`pages_`的映射方向为frame_id->Page*。因此可以通过这两个数据结构分别通过page_id和frame_id获得想要的信息。

## LRU Replacer

frame出现的位置只有两个：free_list和LRU replacer中。当某个frame没有与page绑定时或者删除某个page之后，他就在free_list中；若绑定了page则需要通过LRU replacer对frame对应page中的内容进行替换。我们的LRU replacer包含以下数据结构：

```cpp
  size_t max_num_pages;
  mutex lru_mutex; //不可重入
  list<frame_id_t> lru_cache;
  map<frame_id_t,list<frame_id_t>::iterator> lru_map;
```

其中新加入LRU replacer的frame会加入到lru_cache的头部，替换出去的frame位于lru_cache的尾部。但是有时我们可能需要将lru_cache中间的frame移除LRU replacer，如果从头开始遍历的话就是O(n)的复杂度。我们可以维护一个map记录frame在lru_cache中迭代器的位置，这样就可以通过O(1)的时间查找到该元素完成删除。如果加入新的frame时LRU replacer满了就把最后一个frame移除。

### Pin/Unpin

如果一个进程正在读取一个frame中page的内容，那么应该调用`Pin()`将其移除LRU replacer防止此frame被其他进程使用造成脏读。`Unpin()`即当一个frame暂时不用时放入LRU replacer中供其他进程使用。

```cpp
#include "buffer/lru_replacer.h"

namespace bustub {
    LRUreplacer::LRUreplacer(size_t num_pages) {
        this->max_num_pages = num_pages;
    }

    LRUreplacer::~LRUreplacer() = default;
	
    //传的指针当作返回值使用
    bool LRUreplacer::Victim(frame_id_t *frame_id) {
        lock_guard<mutex> latch(lru_mutex);
        if(lru_map.empty()) {
            lru_mutex.unlock();
            return false; 
        }
        *frame_id = lru_cache.back();
        lru_cache.pop_back();
        lru_map.erase(*frame_id);
        return true;
    }

    void LRUreplacer::Pin(frame_id_t frame_id) {
        lock_guard<mutex> latch(lru_mutex);
        if(lru_map.count(frame_id) != 0){
            lru_cache.erase(lru_map[frame_id]);
            lru_map.erase(frame_id);
        }
    }

    void LRUreplacer::Unpin(frame_id_t frame_id) {
        lock_guard<mutex> latch(lru_mutex);    
        if(lru_map.count(frame_id) != 0){
            return;
        }
        if(this->Size() == this->max_num_pages) {
            lru_map.erase(lru_cache.back());
            lru_cache.pop_back();
        }
        lru_cache.push_front(frame_id);
        lru_map[frame_id] = lru_cache.begin();
    }

    size_t LRUreplacer::Size() { 
        return lru_cache.size(); 
    }

}
```

## Buffer Pool Manager

当我们取页(FetchPageImpl)和新建一个页(NewPageImpl)时，都**首先要从free_list拿frame**与page绑定，**如果free_list空了再去**LRU replacer中寻找一个最近最少未使用的frame用新的page替换掉其绑定的page中的内容(替换page内容前别忘检查是否是脏页，如果是则要刷盘)。很明显直接从free_list中取frame更快，因为不涉及到刷盘和搜索工作。

**frame一直绑定着一个真正的物理页。一个frame就是一个page，一个page也是disk读取来的数据页。判断disk是否为数据页和frame的标准为page_id有无赋值**。frame对应的页内容为空的话就在free_list中，表明没有从disk来的数据页的数据读入frame绑定的物理页中。正因为在free_list中的frame对应的物理页没有内容，因此page_返回的是物理页的指针而不是page_id。

```cpp
//参数是目标
Page *BufferPoolManager::FetchPageImpl(page_id_t page_id) {
  // 1.     Search the page table for the requested page (P).
  // 1.1    If P exists, pin it and return it immediately.
  // 1.2    If P does not exist, find a replacement page (R) from either the free list or the replacer.
  //        Note that pages are always found from the free list first.
  // 2.     If R is dirty, write it back to the disk.
  // 3.     Delete R from the page table and insert P.
  // 4.     Update P's metadata, read in the page content from disk, and then return a pointer to P.
  lock_guard<mutex> latch(latch_);
  frame_id_t fid;

  if(page_table_.count(page_id) != 0) {
    replacer_->Pin(page_table_[page_id]);
    pages_[page_table_[page_id]].pin_count_++;
    return &pages_[page_table_[page_id]];
  }

  if(free_list_.size() != 0) { //先从空闲列表取frame
    fid = free_list_.front();
    free_list_.pop_front();
  } else {
     if(!replacer_->Victim(&fid)) { //free_list满了的去lru获得
       return nullptr;
     }
  }

  Page* page = &pages_[fid];
  if(page->IsDirty()) {
    disk_manager_->WritePage(page->GetPageId(),page->GetData());
  }
  page_table_.erase(page->GetPageId());   //删除旧的页表绑定信息
  
  page_table_[page_id] = fid;
  page->ResetMemory();
  page->page_id_ = page_id;
  page->pin_count_++;
  disk_manager_->ReadPage(page->GetPageId(),page->GetData());
  return page;
}

//参数是返回值
Page *BufferPoolManager::NewPageImpl(page_id_t *page_id) {
  // 0.   Make sure you call DiskManager::AllocatePage!
  // 1.   If all the pages in the buffer pool are pinned, return nullptr.
  // 2.   Pick a victim page P from either the free list or the replacer. Always pick from the free list first.
  // 3.   Update P's metadata, zero out memory and add P to the page table.
  // 4.   Set the page ID output parameter. Return a pointer to P.
  lock_guard<mutex> latch(latch_);
  frame_id_t fid;
  bool all_pinned = true;
  for (frame_id_t i = 0; i < static_cast<frame_id_t>(pool_size_); ++i) {
    if (pages_[i].pin_count_ == 0) {
      all_pinned = false;
      break;
    }
  }
  if (all_pinned) {
    return nullptr;
  }

  if(free_list_.size() != 0) {
    fid = free_list_.front();
    free_list_.pop_front();
  } else if(!replacer_->Victim(&fid)) {
    return nullptr;
  }

  Page* page = &pages_[fid];
  page_id_t new_page_pid = disk_manager_->AllocatePage();
   
  if(page->IsDirty()) {
    disk_manager_->WritePage(page->GetPageId(), page->GetData()); 
  }
  page_table_.erase(page->GetPageId()); //删除旧的页表绑定信息

  page->ResetMemory(); //清空page容器中旧页信息的内容
  page->page_id_ = new_page_pid;
  page->pin_count_++;
  page_table_[page->GetPageId()] = fid;
  *page_id = new_page_pid;
  return page;
}
```

pin_count相当于一个引用计数，只有当pin_count==0时才代表page没有进程读取，此时才可以对该页替换或者删除。当一个页被使用时，我们需要将其pin_count++，UnpinPage时将pin_count--，当pin_count==0时就将这个frame放回LRU replacer。如果我们对page中的内容就行了修改，就需要将其标记为脏页，以便下次替换page时将脏页刷盘。

```cpp
bool BufferPoolManager::UnpinPageImpl(page_id_t page_id, bool is_dirty) { 
  lock_guard<mutex> latch(latch_);
  if(page_table_.count(page_id) == 0) {
    return false;
  }
  Page* p = &pages_[page_table_[page_id]];
  
  if(p->GetPinCount() <= 0) {
    return false; 
  }
  if(is_dirty) {
    p->is_dirty_ = is_dirty;
  } 

  p->pin_count_--;
  if(p->GetPinCount() == 0) {
    replacer_->Unpin(page_table_[page_id]);
  }
  return true;
}
```

若将该页删除时，则需要将page对应的frame放回free_list中并将对应的页表项删除，解除page和frame的绑定关系：

```cpp
bool BufferPoolManager::DeletePageImpl(page_id_t page_id) {
  // 0.   Make sure you call DiskManager::DeallocatePage!
  // 1.   Search the page table for the requested page (P).
  // 1.   If P does not exist, return true.
  // 2.   If P exists, but has a non-zero pin-count, return false. Someone is using the page.
  // 3.   Otherwise, P can be deleted.If P is dirty, write it to disk. Remove P from the page table, reset its metadata and return it to the free list.
  lock_guard<mutex> latch(latch_);
  if(page_table_.count(page_id) == 0) {
    return true;
  } 

  Page* page = &pages_[page_table_[page_id]];
  if(page->GetPinCount() != 0) {
    return false;
  }

  if(page->IsDirty()) {
    disk_manager_->WritePage(page->GetPageId(), page->GetData());
  }
  
  free_list_.push_back(page_table_[page_id]);
  page_table_.erase(page_id);
  page->ResetMemory();
  disk_manager_->DeallocatePage(page_id);
  return true;
}
```

mutex锁是不可重入的，因此不能再其他函数中调用FlushPageImpl和FlushAllPagesImpl两个函数，需要使用disk_manager刷盘：

```cpp
bool BufferPoolManager::FlushPageImpl(page_id_t page_id) {
  // Make sure you call DiskManager::WritePage!
  lock_guard<mutex> latch(latch_);
  if(page_id == INVALID_PAGE_ID || page_table_.count(page_id) == 0) return false;
  disk_manager_->WritePage(page_id, pages_[page_table_[page_id]].GetData());
  return true;
}

void BufferPoolManager::FlushAllPagesImpl() {
  lock_guard<mutex> latch(latch_);
  unordered_map<page_id_t,frame_id_t>::iterator iter;
  for(iter=page_table_.begin(); iter!=page_table_.end(); ++iter) {
    disk_manager_->WritePage(iter->first, pages_[page_table_[iter->first]].GetData());
  }
}
```

