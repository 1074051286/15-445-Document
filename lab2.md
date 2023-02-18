# Lab2

## B+ Tree

对于一个m阶B+树：

- 内部节点若有k个key，则有k+1个指针指向下一个节点。
- 当key的数量大于max_size时要分裂节点，当key的数量小于ceil(max_size/2)时要合并节点。
- 叶子节点分裂时**复制**middle_key到父节点(这个父节点是内部节点)，内部节点分裂时是直接把middle_key推到父节点，**不复制**！
- 要查找的节点全部在叶子节点，叶子节点从大到小排列。
- 内部节点从上往下垂直投影的顺序就是叶子节点从左到右遍历的顺序。

## 1）Insert

插入涉及到三个函数：

- Insert
- StratNewTree
- InsertToLeaf
- InsertToParent

使用者要对B+树插入节点时要调用Insert，此时会分为两种情况：

- 情况一：树没有建立为空树

  此时就要调用StartNewTree()建立一个只有根节点的B+树。我们需要拿到一个新page，将root_page_id设置为这个page的pid，并对其进行初始化：

  ```c++
  INDEX_TEMPLATE_ARGUMENTS
  bool BPLUSTREE_TYPE::Insert(const KeyType &key, const ValueType &value, Transaction *transaction) { 
    {
      const std::lock_guard<std::mutex> guard(root_latch_);
      if(IsEmpty())   {
        StartNewTree(key, value);
        return true;
      }
    }
    return InsertIntoLeaf(key, value, transaction);
  }
  
  //创建新树
  INDEX_TEMPLATE_ARGUMENTS
  void BPLUSTREE_TYPE::StartNewTree(const KeyType &key, const ValueType &value) {
    page_id_t new_pid = INVALID_PAGE_ID;
    Page* new_page = buffer_pool_manager_->NewPage(&new_pid);
    if(new_page == nullptr) {
      throw runtime_error("out of memory");
    }
    
    root_page_id_ = new_pid;
    LeafPage* new_leaf_page = reinterpret_cast<LeafPage*>(new_page->GetData());
    new_leaf_page->Init(root_page_id_, INVALID_PAGE_ID, leaf_max_size_);
    UpdateRootPageId(1); //更新header page(记录元信息)，每次root page变化都要调用此方法
    new_leaf_page->Insert(key, value, comparator_);
    buffer_pool_manager_->UnpinPage(new_pid, true); //此函数修改完毕，pin_count--
  }
  
  INDEX_TEMPLATE_ARGUMENTS
  void B_PLUS_TREE_LEAF_PAGE_TYPE::Init(page_id_t page_id, page_id_t parent_id, int max_size) {
    this->SetPageType(IndexPageType::LEAF_PAGE);
    this->SetSize(0);
    this->SetParentPageId(parent_id);
    this->SetPageId(page_id);
    this->SetMaxSize(max_size);
    this->SetNextPageId(INVALID_PAGE_ID);
  }
  ```

- 情况二：就是树已经建立起来。此时就要判断插入节点后是否需要分裂。如需要分裂则需判断是否会传递到父节点，并且让父节点不断地传递下去，直到找到插入节点后小于max_size的节点或分裂出新的根节点：

  ```c++
  //插入到叶子节点
  INDEX_TEMPLATE_ARGUMENTS
  bool BPLUSTREE_TYPE::InsertIntoLeaf(const KeyType &key, const ValueType &value, 										Transaction *transaction) {
      
    Page* leaf_page = FindLeafPageByOperation(key, Operation::INSERT, transaction);
    LeafPage *leaf_node = reinterpret_cast<LeafPage *>(leaf_page->GetData());  
      
    int size = leaf_node->GetSize();
    int new_size = leaf_node->Insert(key, value, comparator_);
  	
    //如果要插入的key已经存在，失败 
    if (new_size == size) {
      if (root_is_latched) {
        root_latch_.unlock();
      }
      UnlockUnpinPages(transaction);  // 此函数中会释放叶子的所有现在被锁住的祖先（不包括叶子）
      leaf_page->WUnlatch();
      buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), false);  
      return false;
    }
    
    //插入成功且不需要分裂节点，插入成功
    if (new_size < leaf_node->GetMaxSize()) {
      //此处是否需要释放所有祖先节点的锁？
      //似乎不需要，如果叶子节点插入后小于maxsize，说明其父节点之前在findLeafPage已经释放过锁了
      if (root_is_latched) {
        root_latch_.unlock();
      }
      leaf_page->WUnlatch();
      buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), true);  
      return true;
    }
    
    //插入节点后需要分裂，插入成功
    LeafPage *new_leaf_node = Split(leaf_node);  // pin new leaf node
    InsertIntoParent(leaf_node, new_leaf_node->KeyAt(0), new_leaf_node, transaction); 
    leaf_page->WUnlatch();
    buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), true);     
    buffer_pool_manager_->UnpinPage(new_leaf_node->GetPageId(), true); 
    return true;
  }
  
  //插入到父节点，向上传递分裂
  INDEX_TEMPLATE_ARGUMENTS
  void BPLUSTREE_TYPE::InsertIntoParent(BPlusTreePage *old_node, const KeyType &key, 									BPlusTreePage *new_node,Transaction *transaction) {
    if(old_node->IsRootPage()) {
      page_id_t new_pid;
      Page* new_page = buffer_pool_manager_->NewPage(&new_pid);
      if(new_page == nullptr) { 
        throw runtime_error("out of memory");
      }
      InternalPage* new_root_page = reinterpret_cast<InternalPage*>(new_page);
      new_root_page->Init(new_pid, INVALID_PAGE_ID, internal_max_size_);
      UpdateRootPageId(0);
      root_page_id_ = new_pid;
      new_root_page->PopulateNewRoot(old_node->GetPageId(), key, new_node->GetPageId());
      old_node->SetParentPageId(new_pid);
      new_node->SetParentPageId(new_pid);
  
      if(root_is_latched) {
        root_is_latched = false;
        root_latch_.unlock();
      }
      UnlockUnpinPages(transaction);
      buffer_pool_manager_->UnpinPage(new_pid, true);
      return;
    } else {
      page_id_t pid = old_node->GetParentPageId();
      Page* parent = buffer_pool_manager_->FetchPage(pid);
      if(parent == nullptr) { 
        throw runtime_error("out of memory");
      }
      InternalPage* parent_page = reinterpret_cast<InternalPage*>(parent);
      parent_page->InsertNodeAfter(old_node->GetPageId(), key, new_node->GetPageId());
      if(parent_page->GetSize() >= parent_page->GetMaxSize()) {
        InternalPage* splited_page = Split(parent_page); 
        //对于internal_page, 0这个位置的key是直接推到父节点上的，对于自己无效;
        //但是对于leaf_page, 0这个位置的key有效;
        //因此叶子节点是把中间的KEY复制进父节点。而内部节点是把中间的KEY直接推到父节点。
        InsertIntoParent(parent_page, splited_page->KeyAt(0), splited_page); 
        buffer_pool_manager_->UnpinPage(splited_page->GetPageId(), true);
      } else {
        if(root_is_latched) {
          root_is_latched = false;
          root_latch_.unlock();
        }
        UnlockUnpinPages(transaction);  // unlock除了叶子结点以外的所有上锁的祖先节点
      }
      buffer_pool_manager_->UnpinPage(parent->GetPageId(), true);
    }                                       
  }
  
  //分裂函数。
  INDEX_TEMPLATE_ARGUMENTS
  template <typename N>
  N *BPLUSTREE_TYPE::Split(N *old_page) {
    page_id_t new_pid;
    Page* new_page = buffer_pool_manager_->NewPage(&new_pid);
    if(new_page == nullptr) { 
      throw runtime_error("out of memory"); 
    }
  
    if(old_page->IsLeafPage()) {
      LeafPage* new_leaf_page = reinterpret_cast<LeafPage*>(new_page);
      LeafPage* old_leaf_page = reinterpret_cast<LeafPage*>(old_page);
      new_leaf_page->Init(new_pid, old_page->GetParentPageId(), leaf_max_size_);
      old_leaf_page->MoveHalfTo(new_leaf_page);
      new_leaf_page->SetNextPageId(old_leaf_page->GetNextPageId());
      old_leaf_page->SetNextPageId(new_leaf_page->GetPageId());
      return reinterpret_cast<N*>(new_leaf_page);
  
    } else {
      InternalPage* new_leaf_page = reinterpret_cast<InternalPage*>(new_page);
      InternalPage* old_leaf_page = reinterpret_cast<InternalPage*>(old_page);
      new_leaf_page->Init(new_pid, old_page->GetParentPageId(), internal_max_size_);
      
      old_leaf_page->MoveHalfTo(new_leaf_page,buffer_pool_manager_);
      return reinterpret_cast<N*>(new_leaf_page);
    }
    return nullptr;
  }
  ```

## 2）Delete

删除主要包括以下几个函数：

- Remove
- CoalesceOrRedistribute
- Coalesce(合并)
- Redistribute

删除调用的是Remove函数。由于叶子节点在分裂时其middle_key是复制到父节点上，因此如果删除了叶子节点和其父节点共有的key后，会导致父节点的key个数小于min_size(max_size/2)，此时就要将祖先节点(父节点的父节点)的middle_key(祖先节点指向父节点value对应的key)拉下来到父节点上与父节点的兄弟节点进行合并。由于祖先节点也少了一个key，因此同样的可能引发以上操作。所以删除与插入类似，都从叶子节点开始，将删除的影响一步步的向父节点传递。

```c++
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::Remove(const KeyType &key, Transaction *transaction) {
  if(IsEmpty()) {
    return;
  }
  Page* page = FindLeafPageByOperation(key, Operation::FIND, transaction);
  LeafPage* target_page = reinterpret_cast<LeafPage*>(page);
    
  int old_page_size = target_page->GetSize();
  int current_page_size = target_page->RemoveAndDeleteRecord(key, comparator_);
    
  //concurrency control
  if(old_page_size == current_page_size) {
    if(root_is_latched) {
      root_latch_.unlock();
    }
    UnlockUnpinPages(transaction);
    page->WUnlatch();
    buffer_pool_manager_->UnpinPage(page->GetPageId(),false);
    return;
  }

  bool is_deleted = CoalesceOrRedistribute(target_page, transaction);
  if(is_deleted) {
    transaction->AddIntoDeletedPageSet(page->GetPageId());
  }
    
  page->WUnlatch();
  buffer_pool_manager_->UnpinPage(target_page->GetPageId(), false);
  for (page_id_t page_id : *transaction->GetDeletedPageSet()) {
    buffer_pool_manager_->DeletePage(page_id);
  }
  transaction->GetDeletedPageSet()->clear();
}
```

其中CoalesceOrRedistribute函数的作用是判断该page是删除(合并节点)还是保留(向兄弟借key)。如果当前page的key的个数+兄弟节点key的个数之和>=兄弟节点的max_size，那么当前page向兄弟节点借一个key就好；否则就合并节点。

```c++
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::CoalesceOrRedistribute(N *node, Transaction *transaction) {
  if(node->IsRootPage()) {
    bool flag = AdjustRoot(node); 
    if (root_is_latched) {
      root_is_latched = false;
      root_latch_.unlock();
    }
    UnlockUnpinPages(transaction);
    return flag;
  }
  if(node->GetSize() >= node->GetMinSize()) {
    bool flag = AdjustRoot(node); 
    if (root_is_latched) {
      root_is_latched = false;
      root_latch_.unlock();
    }
    UnlockUnpinPages(transaction);
    return flag;
  }

  Page* Parent_page = buffer_pool_manager_->FetchPage(node->GetParentPageId());
  InternalPage* parent = reinterpret_cast<InternalPage*>(Parent_page);
  int index = parent->ValueIndex(node->GetPageId());
  page_id_t sibling_pid = parent->ValueAt(index == 0 ? 1 : index-1);
  Page* sibling_page = buffer_pool_manager_->FetchPage(sibling_pid);
  sibling_page->WLatch();
  N* sibling = reinterpret_cast<N*>(sibling_page);

  if(sibling->GetSize() +node->GetSize() >= sibling->GetMaxSize()) {
    if(root_is_latched) {
      root_is_latched = false;
      root_latch_.unlock();
    }
    Redistribute(sibling, node, index);
    UnlockUnpinPages(transaction);
    sibling_page->WUnlatch();
    buffer_pool_manager_->UnpinPage(parent->GetPageId(), true);
    buffer_pool_manager_->UnpinPage(sibling->GetPageId(), true);
    return false;
  }
  bool is_parent_del = Coalesce(&sibling, &node, &parent, index, transaction);
  if(is_parent_del) {
    transaction->AddIntoDeletedPageSet(parent->GetPageId());
  }
  buffer_pool_manager_->UnpinPage(parent->GetPageId(), true);
  sibling_page->WUnlatch();
  buffer_pool_manager_->UnpinPage(sibling->GetPageId(), true);
  return true;
}
```

**对于重新分配**，由于并没有破坏B+树父子节点的结构，因此不需要判断父节点是否需要合并/借用节点，只要移动节点即可。但是，当把兄弟节点的第一个key移动到目标节点后，父节点中他俩的夹着的那个key一定会发生变化。以删除8为例：

<img src="F:\论文图片\6767.png" alt="6767" style="zoom:50%;" />



<img src="F:\论文图片\6868.png" alt="6767" style="zoom:50%;" />

​		把10拉到兄弟节点第0个位置



<img src="F:\论文图片\6969.png" alt="6767" style="zoom:50%;" />

兄弟节点16所在的位置是无效的：

<img src="F:\论文图片\7070.png" alt="6767" style="zoom:50%;" />

因此我们首先要把当前夹的middle_key放到兄弟节点数组的0位置上并赋值给目标节点，随后我们把兄弟节点0位置的节点删除。此时0位置的key就正好作为新的middle_key，而且由于内部节点0位置的元素是无效的(只是个占位)，所以这个0位置节点的key并不需要移动。

```c++
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
void BPLUSTREE_TYPE::Redistribute(N *neighbor_node, N *node, int index) {
  Page* parent_page = buffer_pool_manager_->FetchPage(node->GetParentPageId());
  InternalPage* parent = reinterpret_cast<InternalPage*>(parent_page);

  if(node->IsLeafPage()) {
    LeafPage* leaf_neighbor = reinterpret_cast<LeafPage*>(neighbor_node);
    LeafPage* leaf_node = reinterpret_cast<LeafPage*>(node);
    if(index == 0) {
      leaf_neighbor->MoveFirstToEndOf(leaf_node);
      parent->SetKeyAt(1, leaf_node->KeyAt(0));
    } else {
      leaf_neighbor->MoveLastToFrontOf(leaf_neighbor);
      parent->SetKeyAt(index, leaf_node->KeyAt(0));
    }
  } else {
    InternalPage* internal_neighbor = reinterpret_cast<InternalPage*>(neighbor_node);
    InternalPage* internal_node = reinterpret_cast<InternalPage*>(node);
    if(index == 0) {
      internal_neighbor->MoveFirstToEndOf(internal_node, parent->KeyAt(1), buffer_pool_manager_);
      parent->SetKeyAt(1, internal_neighbor->KeyAt(0));
    } else {
      internal_neighbor->MoveLastToFrontOf(internal_node, parent->KeyAt(index), buffer_pool_manager_);
      parent->SetKeyAt(index, internal_node->KeyAt(0));
    }
  }
  buffer_pool_manager_->UnpinPage(parent->GetPageId(), true);
}
```

但是对于合并来说，为了保证合并后节点的连续性，我们也需要把父节点的middle_key拉下来合并。这就会导致父节点key个数-1。如果此时父节点key的个数干好为min_size，少一个key后会导致父节点与父节点的兄弟节点合并。如果父节点key的个数大于min_size则没有影响。以下面删除10为例：

<img src="F:\论文图片\71.png" alt="71" style="zoom:50%;" />

删除10后，如果向兄弟节点借key则兄弟节点也小于min_size(2)，此时就要合并。把middel_key(16)拉倒兄弟节点上，并把所有节点拷贝到自己：

<img src="F:\论文图片\72.png" alt="71" style="zoom:50%;" />

修改后发现父节点小于min_size，因此父节点也要进行相同的操作(递归)：

<img src="F:\论文图片\73.png" alt="71" style="zoom:50%;" />

```c++
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::Coalesce(N **neighbor_node, N **node,BPlusTreeInternalPage<KeyType,            page_id_t, KeyComparator> **parent, int index,Transaction *transaction) {
  int new_index = index;
  if(index == 0) {
    std::swap(neighbor_node, node);
    new_index = 1;
  }
  KeyType mid_key = (*parent)->KeyAt(new_index);
  if((*node)->IsLeafPage()) {
    LeafPage* neighbor = reinterpret_cast<LeafPage*>(*neighbor_node);
    LeafPage* self = reinterpret_cast<LeafPage*>(*node);
    self->MoveAllTo(neighbor);
    neighbor->SetNextPageId(self->GetNextPageId());
  } else {
    InternalPage* neighbor = reinterpret_cast<InternalPage*>(*neighbor_node);
    InternalPage* self = reinterpret_cast<InternalPage*>(*node);
    self->MoveAllTo(neighbor, mid_key, buffer_pool_manager_);
  }
  //new_index的value指向删除的节点，因此在父节点中删除并将key保存到子节点。  
  (*parent)->Remove(new_index); 
  return CoalesceOrRedistribute(*parent, transaction);
}
```

