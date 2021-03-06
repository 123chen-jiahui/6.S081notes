## Lab8 Locks

这个实验还是蛮难的...

### 1. Memory allocator(<font color = blue>moderate</font>)

#### 1.1 题目

众所周知，在xv6中，我们申请的内存是在结构体`struct kmem`中申请的，`struct kmem`中的**链表**`struct run *freelist`存放了所有的空闲内存(以page为单位)，在申请和释放内存的时候，可能会存在race condition（其详细解释在课程“lock”一节），为了避免race condition，需要一把锁来保护不变量，即kem中的`struct spinlock lock`，每当要添加元素到freelist或是从freelist删除元素时，都应该acquire这把锁。

但是这又出现问题了，由于频繁地申请和释放空间，所有相关的进程都在**争用同一把锁**，这降低了程序的效率。我们要做的就是解决这个问题，这也是这个lab的主题。

实验给了`user/kalloctest.c`，这个程序大概做的工作是频繁地申请`kalloc`和释放`kfree`内存。最后输出每个锁的争用次数以及总和。

#### 1.2 分析

内存资源(freelist)是全局的，它是对所有CPU都是可见的，为了保护内存资源这个不变量，一个很简单的方法就是这些资源共用一把锁，哪个CPU要修改(申请或释放空间)，它就要先获得这把锁。这也就导致了大量的锁争用，一个解决方法是**分而治之**，既然申请和释放内存是每个CPU的行为，我们可以把内存资源分散到每个CPU上去，即将内存资源分散为`b1, b2, b3, ...bn`等几块，每一块内存资源只能由对应的CPU访问。

这样看来，我们似乎连锁都不需要了，因为资源和CPU已经一一对应了，而每个CPU在任何一个时间都只能做一件事，它要么申请内存，要么释放内存，不可能既申请又释放，这样就不存在race condition，也就不需要锁了。

然而有两个因素(我想到是两个，不一定都对，且可能还有更多)来否定上述想法：

1. 有两种形式**concurrency**，一种是CPU和CPU之间的并发，另一种是CPU和device之间的并发(详见课程“interrupt”一节)，通过分而治之的方法可以排除CPU和CPU之间对同一内存资源的并发访问，但是依然存在后一种并发。虽然我并不知道driven中是否会访问内存，但是毕竟这是一个潜在的隐患。

2. 可能存在这样一种情形：CPU0频繁的申请内存，而其他CPU基本不怎么访问内存，这就导致分配给CPU0的空间被耗尽而分配给其他CPU的空间还有很多剩余的情况，此时若CPU0再申请空间，仅仅报一个`panic("out of memory")`是不可取的，因为明明还存在空闲的内存，只不过在其他CPU的“管辖范围内”。对于这种情况，CPU0需要从其他CPU中偷取空闲内存，这也就导致了CPU对同一资源的并发访问。

综上，由于存在并发，所以每块内存资源都应该有一把锁。但这已远远减少了锁的争用。

#### 1.3 实现

首先，舍弃原来的kmem，为每个CPU划定自己管辖的资源：

```c
struct {
  struct spinlock lock;
  struct run *freelist;
 } kmem[NCPU];
```

修改`kinit`用以初始化

```c
void
kinit()
{
  for (int i = 0; i < NCPU; i ++) {
    initlock(&kmem[i].lock, "kmem");
    // freerange(end, (void *)PHYSTOP); // freerange一遍就好了...
  }
  freerange(end, (void *)PHYSTOP);
}
```

注意本人被kinit坑害过，原因是`freerange`**一次**就行了，也就是说先将所有的内存都放到一个CPU上(不用担心这一点，因为根据上面的分析，资源是可以动态调整的)。

修改`kfree`，将要释放的内存放到当前的CPU上，采用头插法

```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");

  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  push_off();
  int id = cpuid();
  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);
  pop_off();
}
```

调用`cpuid`必须保证关中断，并且在**整个使用周期**都要保持中断关闭(具体我也不知道为什么，但似乎在xv6book的7.4节有讲)。这里说句题外话，从函数开/关中断的函数名看来，中断似乎是以栈的形式存在的，可能是因为存在acquire一把锁但没有release之前，又acquire另一半锁的情况...并且在`sched`函数中可以看到对`nodeoff`进行了检查，nodeoff表示depth of push_off nesting，即中断关闭的深度？扯远了...

接下来是`kalloc`函数，

```c
void *
kalloc(void)
{
  struct run *r;

  push_off();
  int id = cpuid();

  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  else {
    int find = 0;
    for (int i = 0; i < NCPU && find == 0; i ++) {
      if (i != id) {
        acquire(&kmem[i].lock); // 获取另一个cpu的锁
        r = kmem[i].freelist;
        if (r) {
          kmem[i].freelist = r->next; // 从另一个cpu的list上取下来
          r->next = kmem[id].freelist; // 放到这个cpu的list的头部
          find = 1;
        }
        release(&kmem[i].lock);
      }
    }
  }
  release(&kmem[id].lock);
  pop_off();

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  return (void*)r;
}
```

### 2. Buffer cache(<font color = red> hard </font>)

#### 2.1 题目

这题还是太难了...

这道题和前一题的主题是相同的，也是减少锁争用，但是这里的战线转向了`buffer cache`难度立竿见影。

首先简单地介绍一下背景：disk上是以**block**(有时候也称**sector**)为单位来保存数据的，我们知道对disk读写是很慢的，所以如果要对磁盘上的数据进行读写，一般是先将block放到buffer cache中，再进行操作。这一点在计算机中多有应用，比如cache和memory，mmap，还有我们经常用但可能不熟悉的[I/O缓冲区](https://akaedu.github.io/book/ch25s02.html#id2829671)。

```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];
  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  struct buf head;
} bcache;
```

xv6中用`bcache`来表示buffer cache，如上所示。通过**双向循环链表**来组织所有的buf。**源代码里有一个很天才的设计，即LRU算法的实现，但这不是我们讨论的重点**。`bget`函数寻找一个可行的buf，将一个block放到该buf中。由于上述行为可能是并发的，所以需要一把锁来保护不变量，即`bcache.lock`。这就导致了锁的争用。我们要做的就是减少锁的争用

#### 2.2 分析

我们不能像上面那道题一样，通过CPU来划分资源，因为bcache缓冲区真正的在进程（以及CPU）之间共享（**<font color = red>对于内存的分配，当CPU申请了某页内存后，在释放之前，该页内存就为某个进程所私有，一般来说其他进程不能访问该内存；然而buffer cache不同，某个block被放进了buf后，所有进程都可以访问它</font>**）。

好在有`hint`，它告诉了我们怎么做——使用hash表。我想这也是一种分而治之的思想。在原来的实现中，可以发现一个问题：为了找到一个buf，必须遍历整条链表，而为了遍历整条链表，必须获取bcache.lock，这是一把**很大的锁**，而锁越大，串行性越高。hash表将指定编号blockno(关键码key)的block直接映射到某一个`hash bucket`（桶），这就避免了获取大锁在遍历链表寻找的过程。每个hash bucket都有很多buf供映射进来的block使用，将block放到buf中，此时，我们只需要获取这个bucket的锁即可。

同上一题一样，可能会出现桶中buf不够的情形，这时候要从bucket中偷取一个buf，当然偷取的原则是LRU算法。此时需要获取被偷的bucket的锁以保护不变量。

#### 2.3 实现

桶的数据结构：桶内通过链表来组织，需要一个头结点head；同时每个桶须有一把锁

```c
#define NBUCKET 13

struct bucket {
  struct spinlock lock;
  struct buf head;
};

struct {
  struct spinlock lock;
  struct buf buf[NBUF];
  struct bucket hash_table[NBUCKET];
  // Linked list of all buffers, through prev/next.
  // Sorted by how recently the buffer was used.
  // head.next is most recent, head.prev is least.
  // struct buf head;
} bcache;
```

同时在buf中加入成员ticks，用来表示最后被访问的时间，可以以此实现LRU算法

```c
struct buf {
  // 原有的成员
  uint ticks; // 时间戳ticks
};
```

将block放到buf的过程：

1. 给定blockno，通过hash函数（除留余数法）来获取桶号`addr`

2. 获取这个桶的锁

3. 在桶内寻找，查看是否存在buf b满足`b->dev == dev && b->blockno == blockno`，若存在，则这个block已经被映射过了，此时增加其引用次数即可`b->refcnt++`，去第6步。若不存在，去第4步。

4. 仍在这个桶内寻找，查看是否存在空闲的buf b，即`b->refcnt == 0`。这时需要通过LRU算法来寻找，通过时间戳ticks，找到时间戳最小的那个，也就是最早被访问的，若存在，映射此block，去第6步，否则去第5步

5. 遍历其他所有的桶，对于每个桶，都要获取这个桶的锁，然后执行第4步，释放锁。如果能找到满足条件的buf，将这个结点从原来的桶移到addr号桶，这里主要涉及**双向循环链表的插入和删除操作**，然后映射该桶，去第6步；如果找不到这样的buf，panic。**注意**，偷取buf的时候，需要获取被偷取的桶的锁，而下面的代码是直接获取了大锁（bcache.lock），因为那时候我对这个lab还不是很熟悉，属于云里雾里的状态。理论上这两种方法都可以，因为都保护了不变量。但是小锁应该更好。

6. 释放这个桶的锁，然后获取这个buf的睡眠锁acquiresleep

依据此逻辑写出的bget函数如下：

```c
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;
  int addr = blockno % NBUCKET;

  acquire(&bcache.hash_table[addr].lock); // 对当前bucket上锁
  b = bcache.hash_table[addr].head.next;
  while (b != &bcache.hash_table[addr].head) {
    if (b->dev == dev && b->blockno == blockno) {
      b->refcnt ++;
      release(&bcache.hash_table[addr].lock);
      acquiresleep(&b->lock);
      return b;
    }
    b = b->next;
  }
  // printf("%d\n", 1);
  // 整条链都没有找到像样的，因此找这条链中找refcnt==0的结点
  // 原则：LRU(先不考虑将ticks最小的结点放在头部)
  int LRU = 65536;
  int found = 0;
  struct buf *tmp = bcache.hash_table[addr].head.next;
  while (tmp != &bcache.hash_table[addr].head) {
    if (tmp->refcnt == 0 && tmp->ticks < LRU) {
      LRU = tmp->ticks;
      found = 1;
      b = tmp;
    }
    tmp = tmp->next;
  }
  if (found) {
    b->dev = dev;
    b->blockno = blockno;
    b->valid = 0;
    b->refcnt = 1;
    release(&bcache.hash_table[addr].lock);
    acquiresleep(&b->lock);
    return b;
  }
  // 这条链中不存在，要在其他链中偷一个
  // 其他链中不可能存在dev和blockno都相等的结点
  // 所以只需要找refcnt==0，且满足LRU
  // 由于此时要对所有桶遍历，因此需要acquire(&bcache.lock)
  // 这种情况似乎可以和上面的合并
  int new_addr = addr;
  LRU = 65536;
  found = 0;
  acquire(&bcache.lock);
  for (int i = 0; i < NBUCKET; i ++) {
    if (i == addr)
      continue;
    acquire(&bcache.hash_table[i].lock);
    struct buf *tmp = bcache.hash_table[i].head.next;
    while (tmp != &bcache.hash_table[i].head) {
      if (tmp->refcnt == 0 && tmp->ticks < LRU) {
        LRU = tmp->ticks;
        found = 1;
        b = tmp;
        new_addr = i;
      }
      tmp = tmp->next;
    }
    release(&bcache.hash_table[i].lock);
  }
  release(&bcache.lock);

  if (found) {
    // 取下结点，并放到原来那条链上
    acquire(&bcache.hash_table[new_addr].lock);
    b->prev->next = b->next;
    b->next->prev = b->prev;
    release(&bcache.hash_table[new_addr].lock);

    b->next = bcache.hash_table[addr].head.next;
    b->prev = &bcache.hash_table[addr].head;
    bcache.hash_table[addr].head.next->prev = b;
    bcache.hash_table[addr].head.next = b;

    b->dev = dev;
    b->blockno = blockno;
    b->valid = 0;
    b->refcnt = 1;
    release(&bcache.hash_table[addr].lock);
    acquiresleep(&b->lock);
    return b;
  }
  panic("bget: no buffers");
}
```

当一个进程用完了buf以后，需要释放这个buf，这个过程比较简单。将buf的睡眠锁解除，然后减少对该buf的引用（refcnt）。注意要更新buf的时间戳。

```c
brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  int addr = b->blockno % NBUCKET;
  acquire(&bcache.hash_table[addr].lock);
  b->refcnt --;
  b->ticks = ticks;
  release(&bcache.hash_table[addr].lock);
}
```

最后修改一下`bpin`和`bunpin`，虽然我不知道它俩是干嘛的，但是一看就知道需要修改，这一步别忘了。

```c
void
bpin(struct buf *b) {
  int addr = b->blockno % NBUCKET;
  acquire(&bcache.hash_table[addr].lock);
  b->refcnt++;
  release(&bcache.hash_table[addr].lock);
}

void
bunpin(struct buf *b) {
  int addr = b->blockno % NBUCKET;
  acquire(&bcache.hash_table[addr].lock);
  b->refcnt--;
  release(&bcache.hash_table[addr].lock);
}
```

（通过这个实验我才发现自己数据结构的知识是如此薄弱...说实话，读了题目以后，这个hash表和bucket我很久都没弄懂...）

实验至此结束，按照如上代码可以通过所有测试，但是仍有几个问题：

1. 对于桶数的选择，理论上来说，桶数的选择最好是不超过`NBUF`的最大素数，也就是29，但是用29会超时（这也是一个坑点，用了29死活过不了usertests，简直绝望，最后把29改成13就过了）。我想这可能是因为，选择不大于m的最大素数是为了尽可能减小冲突，然而这个问题里面不care冲突，恰恰相反，如果完美地避开了冲突，就说明每个桶只有1个或很少结点，可能造成频繁地从其他桶偷结点，从而拉低了效率。

2. 对buf中时间戳的更新时机。是在bget的时候更新还是在brelse的时候更新？原来的实现是在brelse时实现LRU，但我发现改进后，bget和brelse都可以，似乎不影响。

3. 可以有更好的解决方案，即：每个桶中都按buf的时间戳从小到大排列，这样便于LRU的实现，不过这么做似乎性能上并没有很大提升，并且实现起来有点繁琐。

### 番外

有关`bio.c`中的睡眠锁`sleeplock`。这个东西很牛\*。

且看初始的bget函数和bread函数（即改进之前的版本）:

```c
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached.
  // Recycle the least recently used (LRU) unused buffer.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->valid = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;                                                                                                                                     }
  }
  panic("bget: no buffers");
}
```

```c
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b, 0);
    b->valid = 1;
  }
  return b;
}
```

里面可以发现一个奇怪函数：`acquiresleep`。对如果是获取b->lock，我们是可以理解的，因为这个缓存将会被返回，做下一步操作，所以要获取b->lock，那为什么要以这种奇怪的形式获取呢？因为有两点可以说明我们不能简单得获取自旋锁：

1. 如果两个进程争用一把锁，而磁盘操作可能很耗时，一个进程获取了锁，需要用大量的时间来做磁盘操作，这时另一个进程会一直在那spin，尝试获取锁，这造成了CPU资源的浪费。

2. 由于获取自旋锁的时候会关闭中断，这意味着如果我们只获取自旋锁，我们将收不到任何磁盘产生的中断信息，（磁盘也是设备，它与UART相似，会产生中断，虽然我不知道具体工作原理是什么），所以中断驱动程序就无法运行。

方案：<font color = red>**大锁套小锁**</font>

一个进程<font color = red>**等待**</font>这样一个事件的发生：<font color = red>**大锁没有被锁住**</font>。如果大锁被锁住了（即该block cache的访问权限已经被别的进程抢走了），就sleep，直到那个进程释放大锁并wakeup其他进程。如果大锁没有被锁，置该锁为1（锁住状态），这样就消除了其他进程对该大锁的访问。由于不同进程对大锁的锁住与非锁住状态有并发访问，所以<font color = red>**需要一个小锁来保护这个不变量**</font>。这与我在“sleep&wakeup”中讲到的sleep和wakeup思想是一致的。还有很绝妙的地方是，这么做恰好使得做磁盘操作时，中断处于打开状态，因为acquiresleep返回之前释放了小锁，同时大锁处于锁住状态。

下面来看看这种大锁套小锁——睡眠锁sleeplock：

```c
struct sleeplock {
  uint locked;       // Is the lock held?
  struct spinlock lk; // spinlock protecting this sleep lock

  // For debugging:
  char *name;        // Name of lock.
  int pid;           // Process holding lock
};
```

上面说的大锁就是外层的sleeplock，小锁就是内层的spinlock。每个block cache中都有一个sleeplock。一个有意思的地方是，sleeplock中的locked成员并不是通过acquire或是说test_and_set来设置的，它仅仅是一个condition，而这个condition保证了避免race condition。

接下来是acquiresleep和releasesleep：

```c
void
acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk); // 保护大锁的不变量
  while (lk->locked) {
    sleep(lk, &lk->lk);
  }
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}

void
releasesleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  lk->locked = 0;
  lk->pid = 0;
  wakeup(lk);
  release(&lk->lk);
}
```
