---
title: S6.081 Locks解析
date: 2024-12-23 10:12:15
tags:
categories: S6.081 lab解析
---
### Locks

#### Memory allocktor(moderate)
kalloc中多个cpu进行并发时会产生锁争用，我们要对每一个cpu维护一下freelist来优化，减少锁征用
具体做法是一开始将所有的可用内存分配给一个cpu，然后其他cpu在没有可用内存时可以从其他cpu那里偷一些内存，然后谁释放的内存这个内存就归谁。
##### 修改kmem
```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];
```
##### 修改kinit,给每个cpu分配一个kmem,并初始化自旋锁
```c
void
kinit()
{
  char buffer[100];
  for(int i = 0;i < NCPU;i++){
    snprintf(buffer, 100, "kmem%d", i);
    initlock(&kmem[i].lock, buffer);;
  }
  // initlock(&kmem.lock, "kmem");
  freerange(end, (void*)PHYSTOP);
}
```
##### 修改kfree，获取当前cpu_id，并将内存释放到当前cpu管理的kmem上，在获取cpu_id时需要关闭中断
```c
void
kfree(void *pa)
{
  struct run *r;

  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");
  push_off();
  int cpu_id = cpuid();
  pop_off();
  // Fill with junk to catch dangling refs.
  memset(pa, 1, PGSIZE);

  r = (struct run*)pa;

  acquire(&kmem[cpu_id].lock);
  r->next = kmem[cpu_id].freelist;
  kmem[cpu_id].freelist = r;
  release(&kmem[cpu_id].lock);
}
```
***这里写错了，不应该获取完cpu_id直接打开中断，如果要是这个时候发生时钟中断，它会去运行别的进程，另一个cpu拿到它的话会使得那个cpu操控这个cpu的内存，虽然能正常运行，但是不应该这样***

##### 写一个steal函数，使得当前cpu能够偷其他cpu的内存
```c
void* 
steal(){
  for(int i = 0;i < NCPU;i++){
    acquire(&kmem[i].lock);
    struct run *r = kmem[i].freelist;
    if(r){
      kmem[i].freelist = r->next;
      release(&kmem[i].lock);
      return (void*)r;
    }
    release(&kmem[i].lock);
  }
  return 0;
}
```

##### 最后修改kalloc即可
```c
void *
kalloc(void)
{
  struct run *r;
  push_off();
  int cpu_id = cpuid();
  pop_off();
  acquire(&kmem[cpu_id].lock);
  r = kmem[cpu_id].freelist;
  if(r)
    kmem[cpu_id].freelist = r->next;
  release(&kmem[cpu_id].lock);

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk
  else r = steal();
  return (void*)r;
}
```

#### Buffer cache(hard)
关于这个实验我写了还挺久，一开始写了一个多个静态数量的散列桶，但是每次桶满的时候只会从这个桶里面找到一个块去替换掉它，这样的话还是过不去，可能因为并发量比较大，桶里面数量太少，导致很容易争用锁或者panic，然后改成了多个动态的散列桶，每个桶管理一个链表，当这个里面没有的时候，先看自己桶里面有没有能够替换的块，如果没有的话我们再去遍历其他的桶，然后去偷块即可
##### 修改bcache，改成多个bucket
```c
#define BUCKETSIZE 13
struct bucket
{
  struct spinlock lock;
  struct buf head;
};
struct
{
  struct buf buf[NBUF];
  struct bucket bucket[BUCKETSIZE];
} bcache;
```
##### 修改一下binit,初始化所有bucket
```c
void binit(void)
{
  struct buf *b;
  for (int i = 0; i < BUCKETSIZE; i++)
  {
    initlock(&bcache.bucket[i].lock, "bucket");
    bcache.bucket[i].head.prev = &bcache.bucket[i].head;
    bcache.bucket[i].head.next = &bcache.bucket[i].head;
  }
  for (b = bcache.buf; b < bcache.buf + NBUF; b++)
  {
    b->next = bcache.bucket[0].head.next;
    b->prev = &bcache.bucket[0].head;
    initsleeplock(&b->lock, "buffer");
    bcache.bucket[0].head.next->prev = b;
    bcache.bucket[0].head.next = b;
  }
}
```

##### 写一个get_ticks()，因为获取ticks需要给ticks加锁
```c
uint get_ticks()
{
  uint ans = 0;
  acquire(&tickslock);
  ans = ticks;
  release(&tickslock);
  return ans;
}
```
##### 修改bget，先遍历整个桶，如果没有的话，先看有没有可以替换的block，如果没有的话再从别的块中找，这里去看别的块的时候先把当前这个块的锁给解开，同时持有两个块的锁可能会造成死锁
```c
static struct buf *
bget(uint dev, uint blockno)
{
  struct buf *b;
  int now = blockno % BUCKETSIZE;
  acquire(&bcache.bucket[now].lock);
  struct buf *miss = 0;
  for (b = bcache.bucket[now].head.next; b != &bcache.bucket[now].head; b = b->next)
  {
    if (b->dev == dev && b->blockno == blockno)
    {
      b->refcnt++;
      b->lastuse = get_ticks();
      release(&bcache.bucket[now].lock);
      acquiresleep(&b->lock);
      return b;
    }
    if (b->refcnt == 0 && (miss == 0 || b->lastuse < miss->lastuse))
    {
      miss = b;
    }
  }
  // local lru
  if (miss != 0)
  {
    miss->dev = dev;
    miss->blockno = blockno;
    miss->valid = 0;
    miss->refcnt = 1;
    miss->lastuse = get_ticks();
    release(&bcache.bucket[now].lock);
    acquiresleep(&miss->lock);
    return miss;
  }
  // steal;
  release(&bcache.bucket[now].lock);
  for (int i = (now + 1) % BUCKETSIZE; i != now; i = (i + 1) % BUCKETSIZE)
  {
    acquire(&bcache.bucket[i].lock);
    for (b = bcache.bucket[i].head.next; b != &bcache.bucket[i].head; b = b->next)
    {
      if (b->refcnt == 0 && (miss == 0 || b->lastuse < miss->lastuse))
      {
        miss = b;
      }
    }
    if (miss)
    {
      miss->dev = dev;
      miss->blockno = blockno;
      miss->valid = 0;
      miss->refcnt = 1;
      miss->lastuse = get_ticks();
      miss->prev->next = miss->next;
      miss->next->prev = miss->prev;
      release(&bcache.bucket[i].lock);
      acquire(&bcache.bucket[now].lock);
      miss->next = bcache.bucket[now].head.next;
      miss->prev = &bcache.bucket[now].head;
      bcache.bucket[now].head.next = miss;
      miss->next->prev = miss;
      release(&bcache.bucket[now].lock);
      acquiresleep(&miss->lock);
      return miss;
    }
    release(&bcache.bucket[i].lock);
  }

  panic("bget: no buffers");
}
```
##### 最后就是改一下brelse，把之前的获取全局锁改成获取桶的锁就可以了
```c
void brelse(struct buf *b)
{
  if (!holdingsleep(&b->lock))
    panic("brelse");

  releasesleep(&b->lock);

  acquire(&bcache.bucket[b->blockno % BUCKETSIZE].lock);
  b->refcnt--;
  release(&bcache.bucket[b->blockno % BUCKETSIZE].lock);
}
```