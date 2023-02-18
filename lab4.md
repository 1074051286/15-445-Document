# Lab4

## 1）Lock  Manager

要求实现一个读写锁：

- LockShared：获取读锁；
- LockExclusive：获取写锁
- LockUpgrade：锁升级
- Unlock：释放锁

事务隔离级别对应的锁要求：

- serializable：获取锁，再锁索引；
- rr：可以幻读，不锁索引，事务提交时释放锁；
- read commited：S锁立即释放，不在事务提交时释放锁；
- read uncommited：由于允许脏读，所以允许S和X同时存在，也就**不需要实现S锁**；

**S锁：**由于X锁与S锁和X锁自身都互斥，因此获取锁的准则就是只要是有请求拿到了X锁，那么之后申请X锁的线程就使用条件变量让当前线程wait，直到X锁释放再去获取。只要当前申请的不是X锁，S锁可以一直获得。

**X锁：**如果申请的是X锁的话，需要判断队列前方是否有S锁。如果有S锁还没有获取就要wait。

需要维护的数据结构：

- lock_table：key为rid，保存着rid的condition variable和锁等待队列；
- txn_rid_table_：txn_id->rid，用于**打破死锁**时可以**找到对应的rid进而找到对应的条件变量**来**唤醒rid的请求队列**中wait的请求；
- shared_lock_set/exclusive_lock_set：记录着事务自己获取的锁，方便事务COMMIT/ABORT时统一释放锁

```c++
bool LockManager::LockShared(Transaction *txn, const RID &rid) {
  std::unique_lock<std::mutex> u_lock(latch_);
  if(txn->GetIsolationLevel() == IsolationLevel::READ_UNCOMMITTED) {
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(),AbortReason::LOCKSHARED_ON_READ_UNCOMMITTED);
  }
  //依据2PC，事务在shrinking阶段不能获取锁，如果可以说明是事务里面的代码不对，要ABORT
  if(txn->GetState() == TransactionState::SHRINKING) {
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(),AbortReason::LOCK_ON_SHRINKING);
  } else if(txn->GetState() == TransactionState::ABORTED) {
    return false;
  } 

  if(lock_table_.count(rid) == 0) {
    lock_table_.emplace(std::piecewise_construct, std::forward_as_tuple(rid), std::forward_as_tuple());
  } 
  LockRequest* lck_req = new LockRequest(txn->GetTransactionId(), LockMode::SHARED);
  lck_req->granted_ = false;
  lock_table_[rid].request_queue_.push_back(*lck_req);
  txn->GetSharedLockSet()->emplace(rid);
  //从第一个节点向后搜索到当前tid,如果遇到了X锁，那么就wait直到这个X锁释放并唤醒当前S锁线程执行
  //一个事务只有一把锁(X/S)
  while(IsBlock(txn, rid)) {
    txn_rid_table_.insert({txn->GetTransactionId(), rid});
    lock_table_[rid].cv_.wait(u_lock); //条件变量没有被唤醒就阻塞在这里，X释放锁的时候会唤醒条件变量
      
    //用于死锁破坏后检测事务自身状态从而删除对应LockRqequest
    if(txn->GetState() == TransactionState::ABORTED) {    
      auto lck_req = GetReqInQueue(lock_table_[rid].request_queue_, txn->GetTransactionId());
      lock_table_[rid].request_queue_.erase(lck_req);
      throw TransactionAbortException(txn->GetTransactionId(), AbortReason::DEADLOCK);
    }
  }
  for(auto& req : lock_table_[rid].request_queue_) {
    if(req.txn_id_ == txn->GetTransactionId()) {
      req.granted_ = true;//获得锁
      break;
    }
  }
  return true;
}

bool LockManager::LockExclusive(Transaction *txn, const RID &rid) {
  std::unique_lock<std::mutex> u_lock(latch_);
  if(txn->GetState() == TransactionState::SHRINKING) {
    txn->SetState(TransactionState::ABORTED);
    throw TransactionAbortException(txn->GetTransactionId(),AbortReason::LOCK_ON_SHRINKING);
  } else if(txn->GetState() == TransactionState::ABORTED) {
    return false;
  }
  
  if(lock_table_.count(rid) == 0) {
    lock_table_.emplace(std::piecewise_construct, std::forward_as_tuple(rid), std::forward_as_tuple());
  } 

  LockRequest* lck_req = new LockRequest(txn->GetTransactionId(), LockMode::EXCLUSIVE);
  lck_req->granted_ = false;
  lock_table_[rid].request_queue_.push_back(*lck_req);
  txn->GetExclusiveLockSet()->emplace(rid);//记录事务获得的不同记录的X锁
  
	//前面有S锁的话就得等待S锁获取并释放(释放时唤醒条件变量)才可以获取X锁  
  while(lock_table_[rid].request_queue_.front().txn_id_ != lck_req->txn_id_) {
    txn_rid_table_.insert({txn->GetTransactionId(), rid});
    lock_table_[rid].cv_.wait(u_lock);
    if(txn->GetState() == TransactionState::ABORTED) {
      auto lck_req = GetReqInQueue(lock_table_[rid].request_queue_, txn->GetTransactionId());
      lock_table_[rid].request_queue_.erase(lck_req);
      throw TransactionAbortException(txn->GetTransactionId(), AbortReason::DEADLOCK);
    }
  }
  for(auto& req : lock_table_[rid].request_queue_) {
    if(req.txn_id_ == txn->GetTransactionId()) {
      req.granted_ = true;
      break;
    }
  }
  return true;
}

bool LockManager::IsBlock(Transaction *txn, const RID &rid) {
  bool is_block = false;
  for(LockRequest elem : lock_table_[rid].request_queue_) {
    //搜索到自己就说明前面没有X锁
    if(elem.txn_id_ == txn->GetTransactionId()) {
      break;
    }
    if(elem.lock_mode_ == LockMode::EXCLUSIVE) {
      is_block = true;
      break;
    }
  }
  return is_block; 
}

std::list<LockManager::LockRequest>::iterator LockManager::GetReqInQueue(std::list<LockRequest>& lrq, const txn_id_t &id) {
  for (auto iter = lrq.begin(); iter != lrq.end(); ++iter) {
    if (iter->txn_id_ == id) {
      return iter;
    }
  }
  return lrq.end();
}
```



## 2）Dead Lock Detection

我们需要遍历**lock_table来获取每一个rid对应的request_queue**。request_queue中**获得锁**的事务作为**弧尾**，**没有获得锁**的事务作为**弧头**。

成环检测为基础的dfs，注意维护一个std::vector<txn_id_t> dfs_order记录节点路径。由于是有向图，因此如果发现要加入的节点在dfs_order中已经存在，则说明遇到了环。要注意最新的事务一定是txn_id最大的事务，而且ABORT最新的事务也是成本最小的。

最后破坏死锁时，在把事务状态设置为ABORT后，锁并没有释放。因此要使用条件变量唤醒队列中的LockRequest，去判断对应事务自身状态是否为ABORT。如果是，则在等待队列中把事务对应的LockRequest删除掉并抛出异常。异常会被接受并执行Abort()，释放事务对应的锁来打破死锁状态。

```c++
void LockManager::AddEdge(txn_id_t t1, txn_id_t t2) {
  auto iter = GetTxnIter(t1, t2);
  if(iter != waits_for_[t1].end()) { 
    return; 
  }
  if(waits_for_.count(t1) == 0) {
    waits_for_.insert({t1, std::vector<txn_id_t>()});
  }
  waits_for_[t1].push_back(t2);
  return;
}

void LockManager::RemoveEdge(txn_id_t t1, txn_id_t t2) {
  auto iter = GetTxnIter(t1, t2);
  if(iter != waits_for_[t1].end()) {
    waits_for_[t1].erase(iter);
  }
}


std::vector<std::pair<txn_id_t, txn_id_t>> LockManager::GetEdgeList() { 
  std::vector<std::pair<txn_id_t, txn_id_t>> pairs;
  for(auto iter = waits_for_.begin(); iter != waits_for_.end(); iter++) {
    for(auto tail : waits_for_[iter->first]) {
      pairs.push_back({iter->first, tail});
    }
  }
  return pairs;
 }

void LockManager::RunCycleDetection() {
  while (enable_cycle_detection_) {
    std::this_thread::sleep_for(cycle_detection_interval);
    {
      std::unique_lock<std::mutex> l(latch_);
      waits_for_.clear();
      for(auto iter = lock_table_.begin(); iter != lock_table_.end(); ++iter) {
          //getHead和hetTails是同一个rid里面的lock_request
        for(auto head : GetHeads(iter->second.request_queue_)) {
          for(auto tail : GetTails(iter->second.request_queue_)) {
            if(TransactionManager::GetTransaction(head)->GetState() != TransactionState::ABORTED &&
               TransactionManager::GetTransaction(tail)->GetState()!= TransactionState::ABORTED) {
                 AddEdge(head, tail);
            }
          }
        }
        txn_id_t txn_id;
        if(HasCycle(&txn_id)) {
          Transaction* txn = TransactionManager::GetTransaction(txn_id);
          txn->SetState(TransactionState::ABORTED);        
          RID rid = txn_rid_table_[txn_id];
          lock_table_[rid].cv_.notify_all();
        }
      }
      continue;
    }
  }
}

bool LockManager::HasCycle(txn_id_t *txn_id) { 
  for(auto iter = waits_for_.begin(); iter != waits_for_.end(); ++iter) {
    if(dfs(iter->first)) {
      std::sort(dfs_order_.rbegin(), dfs_order_.rend());
      *txn_id = dfs_order_.front();
      return true;
    }
  }
  return false;
}


bool LockManager::dfs(txn_id_t txn_id) {
  if(!dfs_order_.empty() && IfContains(txn_id)) {
      return true;
  }
  dfs_order_.push_back(txn_id);
  std::sort(waits_for_[txn_id].begin(), waits_for_[txn_id].end());
  for(txn_id_t tid : waits_for_[txn_id]) {
    if(dfs(tid)) {
      return true;
    }
  }
  return false;
}

std::vector<txn_id_t>::iterator LockManager::GetTxnIter(txn_id_t t1, txn_id_t t2) {
  for(auto iter = waits_for_[t1].begin(); iter != waits_for_[t1].end(); ++iter) {
    if(*iter == t2) {
      return iter;
    }
  }
  return waits_for_[t1].end();
}

bool LockManager::IfContains(txn_id_t txn_id) {  
  for(txn_id_t tid : dfs_order_) {
      if(tid == txn_id) {
        return true;
      }
    }
    return false;
}

std::vector<txn_id_t> LockManager::GetHeads(std::list<LockRequest> reqs) {
  std::vector<txn_id_t> res;
  for(auto req : reqs) {
    if(!req.granted_) {
      break;
    }
    res.push_back(req.txn_id_); //grant==true分配锁的加入头队列
  }
  return res;
}

std::vector<txn_id_t> LockManager::GetTails(std::list<LockRequest> reqs) {
  std::vector<txn_id_t> res;
  for(auto req : reqs) {
    if(!req.granted_) {
      res.push_back(req.txn_id_);//grant==false未分配锁的加入尾队列
    } 
  }
  return res;
}
```

