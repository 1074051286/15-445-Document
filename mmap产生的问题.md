# 为什么mmap替代不了buffer pool

## mmap流程：

- 分配一块虚拟内存空间并返回指向这段虚拟内存空间的指针，此时分配的虚拟内存空间不进行任何操作；
- 程序用分配的指针去访问虚拟内存，由于页表中没有这段虚拟内存地址的映射，引发缺页异常；
- OS从磁盘中调取文件放入page cache中并在页表中建立虚拟地址到page cache物理地址的映射，随后将页表项放入TLB缓存；

## mmap在DBMS中的诱惑

- 不需要记录那些page被修改、那些page在内存中等信息，page管理全部交个OS；
- 少了一次page cache到user space的拷贝，节省了空间，提高了访问速度；
- 指针操作，就像访问内存数据一样，方便快捷；

## 不使用mmap的理由

### 事务安全

由于OS没有事务的概念，很可能在一个事务没有提交时就把脏页刷盘了。一般采取主备思想处理事务问题：

- **OS copy-on-write：**使用mmap建立两个映射，一个映射作为primary space，另一个映射使用MAP_PRIVATE(开启CoW)创建为work space。事务没提交时的修改统统在workspace进行，由于有cow，OS会重新分配一块物理内存空间将要修改的内容拷贝过来进行修改。当事务提交WAL落盘后，再将修改应用到parimary space中刷到磁盘；
- **user space copy-on-write：**事务没提交前先把要修改的页放到用户空间自己维持的buffer中进行修改，等事务提交WAL落盘后再把修改应用到mmap映射的虚拟内存上；
- **shadow paging**

### IO停顿

- **mmap不支持异步读取**，读取大量数据时会造成阻塞；
- 由于mmap的刷页是透明的，因此想要读取的页可能不在page cache中，造成中断处理阻塞线程。可以使用`mlock`来pin住将来要重复查询的页面，但是每个进行可以pin住的page是有限的，由于page cache大小有限，如果pin的page多了的时候会影响其他进程甚至OS本身的运行。更离谱的是即使pin住了根据POSIX协议OS也可以刷新脏页，因此pin住了也并不安全；
- 或者根据查询的特点使用`madvise`改变OS的存取页模式。但是相对于`mlock`来说控制粒度太大且OS可能会忽略建议；

### 错误处理

- OS自动刷页，无法使用checksum检查页的数据完整性；
- 每次使用mmap都会产生SIGBUS，异常处理代码会遍地飞。使用buffer pool的话可以很好把错误处理集中在buffer pool层中；

### 性能问题

mmap的性能问题主要出现在**page cache满了之后**上，原因如下：

- OS只是用**一个线程执行页面替**换，没有充分利用多核CPU的处理能力；
- 页表会被锁保护，换页调页会造成页表的并发修改，**竞争页表上的锁**会浪费时间；
- **TLB shootdown**。TLB是每个CPU一个且内容相同。删除自己本地CPU的entry是很快的，但是通知到别的CPU核心删除自己TLB中对应的entry需要占用数千个处理器周期。
