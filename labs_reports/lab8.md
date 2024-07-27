# Lab:locks

## 实验目的

重新设计代码以提高并行性

## 实验过程

## Memory allocator

**要求**：通过修改内存分配器的设计，以减少锁竞争，从而提高多核系统中的性能。具体来说，需要实现每个CPU都有一个独立的自由列表（free list），每个列表都有自己的锁。这样可以让不同CPU上的内存分配和释放操作可以并行进行，从而减少锁的争用。还需要实现当一个CPU的自由列表为空时，能够从其他CPU的自由列表中获取部分内存。

1. 首先在`kernel/kalloc.c`中，为每个CPU分配`kmem`

   ```c
   struct {
     struct spinlock lock;
     struct run *freelist;
   } kmem[NCPU]; // each cpu has one 
   ```

2. 修改kinit

   ```c
   void
   kinit()
   {
     char lockname[10];
     for (int i = 0; i < NCPU; i++){
       initlock(&kmem[i].lock, "kmem");
     }
     freerange(end, (void*)PHYSTOP);
   }
   ```

3. 修改kfree。记得查看cpuid前先关中断。

   ```c
     r = (struct run*)pa;
   
     push_off(); // need tu interrupt before get cpuid()
     int id = cpuid();
     acquire(&kmem[id].lock);
     r->next = kmem[id].freelist;
     kmem[id].freelist = r;
     release(&kmem[id].lock);
     pop_off();
   }
   ```

4. 修改kalloc，偷取。记得查看cpuid前先关中断。

   ```c
   {
     struct run *r;
   
     push_off();
     int id = cpuid();
     acquire(&kmem[id].lock);
     r = kmem[id].freelist;
     if(r)
       kmem[id].freelist = r->next;
     else {
       int new_id;
       for (new_id = 0; new_id < NCPU; ++new_id){
         if (new_id == id)
           continue;
         acquire(&kmem[new_id].lock);
         r = kmem[new_id].freelist;
         if (r){
           kmem[new_id].freelist = r->next;
           release(&kmem[new_id].lock);
           break;
         }
         release(&kmem[new_id].lock);
       }
     }
     release(&kmem[id].lock);
     pop_off();
   
     if(r)
   ```

   ![image-20240719112833940](.\images\image-20240719112833940.png)

   ### Buffer cache

   **要求**：优化xv6操作系统中的缓冲区缓存（buffer cache），减少多个进程之间对缓冲区缓存锁的争用，从而提高系统的性能和并发能力。

   1. 先定义`buckets`数量（在`kernel/bio.c`），并添加到`bcache`中。同时实现一个简单的hash

      ```c
      #define NBUCKETS 13
      
      struct {
        struct spinlock lock[NBUCKETS];
        struct buf buf[NBUF];
      
        // Linked list of all buffers, through prev/next.
        // Sorted by how recently the buffer was used.
        // head.next is most recent, head.prev is least.
        struct buf head[NBUCKETS];
      } bcache;
      
      int hash(uint blockno){
        return blockno % NBUCKETS;
      }
      ```

   2. 记得在`def.h`中声明`hash`函数

      ```c
      int             hash(uint id);
      ```

   3. 接下来修改`bcache`相关函数

      ```c
      void
      binit(void)
      {
        struct buf *b;
      
        for (int i = 0; i < NBUCKETS; i++){
          initlock(&bcache.lock[i], "bcache"); 
        }
      
        // Create linked list of buffers
        for (int i = 0; i < NBUCKETS; i++){
          bcache.head[i].prev = &bcache.head[i];
          bcache.head[i].next = &bcache.head[i];
        }
        for(b = bcache.buf; b < bcache.buf+NBUF; b++){
          b->next = bcache.head[0].next;
          b->prev = &bcache.head[0];
          initsleeplock(&b->lock, "buffer");
          bcache.head[0].next->prev = b;
          bcache.head[0].next = b;
        }
      }
      ```

      修改`bget`函数

      ```c
      static struct buf*
      bget(uint dev, uint blockno)
      {
        struct buf *b;
      
        int id = hash(blockno);
      
        acquire(&bcache.lock[id]);
      
        // Is the block already cached?
        for(b = bcache.head[id].next; b != &bcache.head[id]; b = b->next){
          if(b->dev == dev && b->blockno == blockno){
            b->refcnt++;
            release(&bcache.lock[id]);
            acquiresleep(&b->lock);
            return b;
          }
        }
          
        // Not cached.
        // Recycle the least recently used (LRU) unused buffer.
        int i = id;
        while (1){
          i = (i + 1) % NBUCKETS;
          if (i == id) continue;
          acquire(&bcache.lock[i]);
          for(b = bcache.head[i].prev; b != &bcache.head[i]; b = b->prev){
            if(b->refcnt == 0) {
              b->dev = dev;
              b->blockno = blockno;
              b->valid = 0;
              b->refcnt = 1;
      
              // disconnect
              b->prev->next = b->next;
              b->next->prev = b->prev;
      
              release(&bcache.lock[i]);
      
              b->prev = &bcache.head[id];
              b->next = bcache.head[id].next;
              b->next->prev = b;
              b->prev->next = b;
              release(&bcache.lock[id]);
              acquiresleep(&b->lock);
              return b;
            }
          }
          release(&bcache.lock[i]);
        }
        panic("bget: no buffers");
      }
      ```

      修改`brelse`函数

      ```c
      void
      brelse(struct buf *b)
      {
          if(!holdingsleep(&b->lock))
          panic("brelse");
      
        int id = hash(b->blockno);
      
        releasesleep(&b->lock);
      
        acquire(&bcache.lock[id]);
        b->refcnt--;
        if (b->refcnt == 0) {
          // no one is waiting for it.
          b->next->prev = b->prev;
          b->prev->next = b->next;
          b->next = bcache.head[id].next;
          b->prev = &bcache.head[id];
          bcache.head[id].next->prev = b;
          bcache.head[id].next = b;
        }
      
        release(&bcache.lock[id]);
      }
      ```

      修改`bpin`和`bunpin`

      ```c
      void
      bpin(struct buf *b) {
        int id = hash(b->blockno);
        acquire(&bcache.lock[id]);
        b->refcnt++;
        release(&bcache.lock[id]);
      }
      
      void
      bunpin(struct buf *b) {
        int id = hash(b->blockno);
        acquire(&bcache.lock[id]);
        b->refcnt--;
        release(&bcache.lock[id]);
      }
      ```

      ![image-20240719114300999](.\images\image-20240719114300999.png)

## 实验中遇到的问题及解决方案

链表的使用需要比较细心的推导和逻辑关系

**问题**：usertests中出现panic: balloc: out of blocks错误

**解决方案**：修改`param.h`中的`FSSZIE 1000`为`FSSZIE 10000`

## 实验心得

在开始实验之前，我深入理解了 Buffer Cache 的结构，包括缓存的大小、缓存块的管理方式以及数据结构等。这为我后续的编码和设计提供了重要的指导。实验要求的功能涵盖了从缓存的获取、写入到缓存的释放等多个方面。我将实验任务分成不同的阶段，逐步实现和测试每个功能。这样有助于确保每个功能都能够独立正常运行。在多线程环境下，缓存的并发访问可能导致竞态条件等问题。我使用了适当的同步机制，如自旋锁和信号量，以确保缓存的安全性和一致性。