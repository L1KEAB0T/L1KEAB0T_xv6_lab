# Lab5: Copy-on-Write Fork for xv6

## 实验目的

xv6中的`fork()`系统调用将父进程的所有用户空间内存复制到子进程中。如果父进程较大，则复制可能需要很长时间。更糟糕的是，这项工作经常造成大量浪费；例如，子进程中的`fork()`后跟`exec()`将导致子进程丢弃复制的内存，而其中的大部分可能都从未使用过。另一方面，如果父子进程都使用一个页面，并且其中一个或两个对该页面有写操作，则确实需要复制。

COW fork()的目标是推迟到子进程实际需要物理内存拷贝时再进行分配和复制物理内存页面。当我们创建子进程时，与其创建，分配并拷贝内容到新的物理内存，其实我们可以直接共享父进程的物理内存page。所以这里，我们可以设置子进程的PTE指向父进程对应的物理内存page。

## 实验过程

1. 修改`uvmcopy()`**将父进程的物理页映射到子进程**，而不是分配新页。

   ```c
   #define PTE_F (1L << 8) // cow的fork
   ```

   ​	在`kernel/kalloc.c`中定义INDEX宏，用物理地址除4096（即右移12位）实现定位

   ```c
   #define INDEX(pa) (((char*)pa - (char*)PGROUNDUP((uint64)end)) >> 12)
   ```

   修改`kmem`结构，增加计数

   ```c
   struct {
     struct spinlock lock;
     struct run *freelist;
     // add ref
     struct spinlock ref_lock;
     uint *ref_count;
   } kmem;
   ```

   最后定义一些`def.h`中声明所需函数，这些函数的主要目的是抽象计数的相关操场

   ```c
   int             get_kmem_ref(void *pa);
   void            add_kmem_ref(void *pa);
   void            acquire_ref_lock();
   void            release_ref_lock();
   ```

   并在`kernel/kalloc.c`实现它们

   ```c
   int get_kmem_ref(void *pa){
     return kmem.ref_count[INDEX(pa)];
   }
   
   void add_kmem_ref(void *pa){
     kmem.ref_count[INDEX(pa)]++;
   }
   
   void acquire_ref_lock(){
     acquire(&kmem.ref_lock);
   }
   
   void release_ref_lock(){
     release(&kmem.ref_lock);
   }
   ```

2. 修改`uvmcopy`，不为子进程分配内存，而是使父子进程共享内存，但禁用`PTE_W`，同时标记`PTE_F`

   ```c
     //vm.c
     for(i = 0; i < sz; i += PGSIZE){
       if((pte = walk(old, i, 0)) == 0)
         panic("uvmcopy: pte should exist");
       if((*pte & PTE_V) == 0)
         panic("uvmcopy: page not present");
       // set unwrite and set rsw
       *pte = ((*pte) & (~PTE_W)) | PTE_RSW;
       pa = PTE2PA(*pte);
       flags = PTE_FLAGS(*pte);
       // stop copy
       // if((mem = kalloc()) == 0)
       //   goto err;
       // memmove(mem, (char*)pa, PGSIZE);
       if(mappages(new, i, PGSIZE, pa, flags) != 0){
         // kfree(mem);
         goto err;
       }
       add_kmem_ref((void*)pa);
     }
     return 0;
   ```

   修改`usertrap`。由于cow中只有写页面错误，只需要考虑对应`rscause=15`的情况。当cow页面中出现`page fault`，释放新页面，复制旧页面到新页面中，并且设新的PTE的`PTE_W`位为`true`

   ```c
   //trap.c
     else if (r_scause() == 15) { // write page fault
       uint64 va = PGROUNDDOWN(r_stval());
       pte_t *pte;
       if (va > p->sz || (pte = walk(p->pagetable, va, 0)) == 0){
         p->killed = 1;
         goto end;
       }
   
       if (((*pte) & PTE_RSW) == 0 || ((*pte) & PTE_V) == 0 || ((*pte) & PTE_U) == 0){
         p->killed = 1;
         goto end;
       }
   
       uint64 pa = PTE2PA(*pte);
       acquire_ref_lock();
       uint ref = get_kmem_ref((void*)pa);
       if (ref == 1){
         *pte = ((*pte) & (~PTE_RSW)) | PTE_W;
       }
       else {
         char* mem = kalloc();
         if (mem == 0){
           p->killed = 1;
           release_ref_lock();
           goto end;
         }
   
         memmove(mem, (char*)pa, PGSIZE);
         uint flag = (PTE_FLAGS(*pte) | PTE_W) & (~PTE_RSW);
         if (mappages(p->pagetable, va, PGSIZE, (uint64)mem,flag) != 0){
           kfree(mem);
           p->killed = 1;
           release_ref_lock();
           goto end;
         }
         kfree((void*)pa);
       }
       release_ref_lock();
     }
     else if((which_dev = devintr()) != 0){
       // ok
     } else {
       printf("usertrap(): unexpected scause %p pid=%d\n", r_scause(), p->pid);
       printf("            sepc=%p stval=%p\n", r_sepc(), r_stval());
       p->killed = 1;
     }
   ```

   walk是外部函数，因此还需要在`kernel/trap.c`中添加外部声明

   ```c
   extern pte_t* walk(pagetable_t, uint64, int);
   ```

3. 修改`mappages`，避免其在PTE_V非法时panic

   ```c
   //vm.c  
     for(;;){
       if((pte = walk(pagetable, a, 1)) == 0)
         return -1;
       // if(*pte & PTE_V)
       //   panic("remap");
       *pte = PA2PTE(pa) | perm | PTE_V;
       if(a == last)
         break;
   ```

4. 先修改`kinit`：

   ```c
   void
   kinit()
   {
     initlock(&kmem.lock, "kmem");
     initlock(&kmem.ref_lock, "ref");
   
     //total physical pages
     uint64 physical_pages = ((PHYSTOP - (uint64)end) >> 12) + 1;
     physical_pages = ((physical_pages * sizeof(uint)) >> 12) + 1;
     kmem.ref_count = (uint*) end;
     uint64 offset = physical_pages << 12;
     freerange(end + offset, (void*)PHYSTOP);
   }
   ```

   修改`freerange`，初始化计数

   ```c
   void
   freerange(void *pa_start, void *pa_end)
   {
     char *p;
     p = (char*)PGROUNDUP((uint64)pa_start);
     for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE){
       kmem.ref_count[INDEX((void*)p)] = 1;
       kfree(p);
     }
   }
   ```

   最后修改`kfree`，计数为0后才释放

   ```c
     if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
       panic("kfree");
   
     // check if the number reaches 0
     acquire(&kmem.lock);
     if (--kmem.ref_count[INDEX(pa)]){
       release(&kmem.lock);
       return;
     }
     release(&kmem.lock);
   
     // Fill with junk to catch dangling refs.
     memset(pa, 1, PGSIZE);
   ```

   修改`kalloc`，计数设为1

   ```c
     acquire(&kmem.lock);
     r = kmem.freelist;
     if(r){
       kmem.freelist = r->next;
       kmem.ref_count[INDEX((void*)r)] = 1;
     }
     release(&kmem.lock);
   ```

   修改`copyout`，操作类似`usertraps`

   ```c
   copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
   {
     uint64 n, va0, pa0;
     pte_t* pte;
   
     while(len > 0){
       va0 = PGROUNDDOWN(dstva);
       if(va0 >= MAXVA)  
         return -1;
       if((pte = walk(pagetable, va0, 0)) == 0)
         return -1;
       if (((*pte & PTE_V) == 0) || ((*pte & PTE_U)) == 0) 
         return -1;
   
       pa0 = PTE2PA(*pte);
       if (((*pte & PTE_W) == 0) && (*pte & PTE_RSW)){
         acquire_ref_lock();
         if (get_kmem_ref((void*)pa0) == 1) {
             *pte = (*pte | PTE_W) & (~PTE_RSW);
         }
         else {
           char* mem = kalloc();
           if (mem == 0){
             release_ref_lock();
             return -1;
           }
           memmove(mem, (char*)pa0, PGSIZE);
           uint new_flags = (PTE_FLAGS(*pte) | PTE_RSW) & (~PTE_W);
           if (mappages(pagetable, va0, PGSIZE, (uint64)mem, new_flags) != 0){
             kfree(mem);
             release_ref_lock();
             return -1;
           }
           kfree((void*)pa0);
         }
         release_ref_lock();
       }
   
       pa0 = walkaddr(pagetable, va0);
       if(pa0 == 0)
         return -1;
   ```

   ![image-20240718113833041](.\images\image-20240718113833041.png)

   ![image-20240718113901424](.\images\image-20240718113901424.png)

## 实验中遇到的问题以及解决方案

**问题**：对于位移操作和位操作理解问题

**位操作 (`& ~`) 的解析**：

在 xv6 的页表管理中，通常使用位操作来设置或清除特定的标志。我们来看一个具体的例子：

```
c
复制代码
*pte = ((*pte) & ~PTE_W) | PTE_RSW;
```

这里的 `*pte` 是一个页表项，`PTE_W` 和 `PTE_RSW` 是位标志。

- `*pte & ~PTE_W`：
  - `PTE_W` 是一个位掩码，例如，假设它的值是 `0x2`，二进制表示为 `00000010`。
  - `~PTE_W` 是按位取反操作，结果是 `11111101`。
  - `*pte & ~PTE_W` 的效果是清除 `*pte` 中的写标志位。
- `| PTE_RSW`：
  - `PTE_RSW` 是另一个位标志，例如，假设它的值是 `0x200`，二进制表示为 `1000000000`。
  - `*pte & ~PTE_W | PTE_RSW` 的效果是首先清除写标志位，然后设置 `PTE_RSW` 标志位。

总的来说，这段代码的效果是清除页表项中的写标志位，并设置 `PTE_RSW` 标志位。

**位移操作 (`<< 12`) 的解析**：

在计算机系统中，页表管理通常以页为单位，而页的大小通常是 `4096` 字节（也就是 `2^12` 字节）。将虚拟地址映射到物理地址时，需要计算页的编号和页内偏移。

- `va >> 12`：
  - `va` 是一个虚拟地址。
  - 右移 `12` 位的操作将虚拟地址除以 `4096`，得到页号。
- `pa << 12`：
  - `pa` 是一个物理页帧号。
  - 左移 `12` 位的操作将页帧号乘以 `4096`，得到页的基地址。

举个例子，假设一个虚拟地址 `va = 0x12345`：

- 计算页号：`0x12345 >> 12` 等于 `0x12`。
- 页内偏移：`0x12345 & 0xFFF` 等于 `0x345`。

所以，虚拟地址 `0x12345` 对应的页号是 `0x12`，页内偏移是 `0x345`。

## 实验心得

在完成 Copy-on-Write Fork for xv6 实验过程中，我深刻理解了操作系统内存管理中的核心概念，特别是位操作和位移操作的精妙之处。通过实现 COW 机制，我学会了如何在页表项中设置和清除标志位，以及如何高效地处理页的分配和复制，进一步增强了我对操作系统底层细节和内存优化技术的认识。这次实验不仅提升了我的编程技能，也加深了我对内核设计与实现的理解。