# Lab3:page tables

## 实验目的

探索页表的相关，主要是修改以简化从用户空间拷贝到内核空间的操作

## 实验过程

### Speed up system calls

**要求**：通过页表来加速系统调用

1. porc 结构是内核态数据，用户态无法直接读取，因此需要通过系统调用 getpid 来读到 pid。这里需要我们创建一个共享 PTE，将虚拟地址 USYSCALL 映射到 pid 的物理地址，这样用户态直接读取 USYSCALL 就可以获取到 pid 了。

   首先，我们 proc 新增一个字段，存放 usyscall 的地址：

   ```c
   // kernel/proc.h
   struct proc {
    // ...  
    struct usyscall *usyscall;
   }
   12345
   ```

   在 allocproc 时，为其分配一块物理内存，分配方式可参考 trapframe 的分配。

   ```c
   // kernel/proc.c
   static struct proc*
   allocproc(void)
   {
     // ...
     // Allocate a trapframe page.
     if((p->trapframe = (struct trapframe *)kalloc()) == 0){
       freeproc(p);
       release(&p->lock);
       return 0;
     }
   
     // lab3.1 alloc usyscall
     if((p->usyscall = (struct usyscall *)kalloc()) == 0){
       freeproc(p);
       release(&p->lock);
       return 0;
     }
     // ...
   }
   ```

2. proc 有一个字段名为 pagetable，它就是一个 uint64，存放页表的地址。页表的初始化位于函数 proc_pagetable 中，通过 mappages 在页表中注册新的 PTE，参考 TRAPFRAME 的方式，将 USYSCALL 映射到 p->usyscall 中。注意映射之前要先赋值 p->usyscall->pid = p->pid。

   ```c
   // kernel/proc.c
   pagetable_t
   proc_pagetable(struct proc *p)
   {
     pagetable_t pagetable;
     // ...
     // map the trapframe page just below the trampoline page, for
     // trampoline.S.
     if(mappages(pagetable, TRAPFRAME, PGSIZE,
                 (uint64)(p->trapframe), PTE_R | PTE_W) < 0){
       uvmunmap(pagetable, TRAMPOLINE, 1, 0);
       uvmfree(pagetable, 0);
       return 0;
     }
   
     // lab3, map one read-only page at USYSCALL, store the PID
     p->usyscall->pid = p->pid;
     if(mappages(pagetable, USYSCALL, PGSIZE,(uint64)p->usyscall, PTE_R | PTE_U) < 0){
       uvmunmap(pagetable, USYSCALL, 1, 0);
       uvmfree(pagetable, 0);
     }
   
     return pagetable;
   }
   ```

   flag 位不仅仅要置位 PTE_R，还要置位 PTE_U。只有置位了 PTE_U 的页，用户态才有权访问，否则只能 supervisor mode 才能访问。RISC-V 一共有三个特权级，如下：

   | Level | Encodeing | 特权级           | 缩写 | 权限                       |
   | ----- | --------- | ---------------- | ---- | -------------------------- |
   | 0     | 00        | User/Application | U    | 运行用户程序               |
   | 1     | 01        | Supervisor       | S    | 运行操作系统内核和驱动     |
   | 2     | 10        | -                | -    |                            |
   | 3     | 11        | Machine          | M    | 运行 BootLoader 和其他固件 |

3. 新增完映射后，我们需要在进程 free 时对其解映射，位于函数 proc_freepagetable 中，可以参考 TRAPFRAME 的解映射。

   ```c
   // kernel/proc.c
   void
   proc_freepagetable(pagetable_t pagetable, uint64 sz)
   {
     uvmunmap(pagetable, TRAMPOLINE, 1, 0);
     uvmunmap(pagetable, TRAPFRAME, 1, 0);
     uvmunmap(pagetable, USYSCALL, 1, 0);
     uvmfree(pagetable, sz);
   }
   ```

4. 在 freepagetable 释放给 usyscall 分配的物理内存

   ```c
   // kernel/proc.c
   static void
   freeproc(struct proc *p)
   {
     if(p->usyscall)
       kfree((void*)p->usyscall);
     p->usyscall = 0;
   }
   ```

   ![image-20240717145632738](.\images\image-20240717145632738.png)

## Print a page table

**要求**：实现一个内核函数 vmprint，其接收一个 pagetable，能够将其中所有的可用 PTE 的信息全部打印出来。

1. 和虚拟地址有关的内核函数位于 `kernel/vm.c` 中，因此 vmprint 也放在里面，代码如下，注意一下类型转换。

   ```c
   // kernel/vm.c
   void 
   vmprint(pagetable_t pagetable)
   {
     printf("page table %p\n", pagetable);
     for(int i = 0; i < 512; i++){
       pte_t pte = pagetable[i];
       if(pte & PTE_V){
         printf("..%d: pte %p pa %p\n", i, pte, PTE2PA(pte));
         pagetable_t second = (pagetable_t)PTE2PA(pte);
         for(int j = 0; j < 512; j++){
           pte = second[j];
           if(pte & PTE_V){
             printf(".. ..%d: pte %p pa %p\n", j, pte, PTE2PA(pte));
             pagetable_t third = (pagetable_t)PTE2PA(pte);
             for(int k = 0; k < 512; k++){
               pte = third[k];
               if(pte & PTE_V){
                 printf(".. .. ..%d: pte %p pa %p\n", k, pte, PTE2PA(pte));
               }
             }
           }
         }
       }
     }
   }
   ```

2. 接下来只需要在 exec 中插桩即可，当且仅当 pid 为 1 才调用 vmprint。

   ```c
   // kernel/exec.c
   int
   exec(char *path, char **argv)
   {
     // ...
     // lab3.2
     if(p->pid == 1){
       vmprint(p->pagetable);
     }
     
     return argc; // this ends up in a0, the first argument to main(argc, argv)
     // ...
   }
   
   ```

3. 在def.h中声明，这样可以在exec.c中调用

   ```c
   // add vmprint
   void            vmprint(pagetable_t pagetable);
   ```

![image-20240717152400326](.\images\image-20240717152400326.png)

### Detecting which pages have been accessed

**要求**：要求实现一个系统调用 sys_pgaccess，其会从一个虚拟地址对应的 PTE 开始，往下搜索一定数量的被访问（read/write）过的页表，并把结果通过 mask 的方式返回给用户。每当 sys_pgacess 调用一次，页表被访问标志就要清 0。

1. 页表被访问可以通过每个 PTE 有个 PTE_A 位，该位被置 1 则说明被访问过，该位被置 0 则说明没被访问过。

   通过虚拟地址依次遍历后续 PTE通过 walk 可得到虚拟地址对应的 PTE。其次，PTE 是连续的，那么对应的虚拟地址也应是连续的。最后，一个 PTE 大小为 PGSIZE，因此只要将虚拟地址按 PGSIZE 累加即可得到后续的 PTE。

2. 在 `kernel/riscv.h` 中定义相关的宏：

   ```c
   // kernel/riscv.h
   #define PTE_A (1L << 6)
   ```

3. 接下来实现 sys_pgaccess，其接收三个参数，分别为：1. 起始虚拟地址；2. 遍历页数目；3. 用户存储返回结果的地址。因为其是系统调用，故参数的传递需要通过 argaddr、argint 来完成。

   通过不断的 walk 来获取连续的 PTE，然后检查其 PTE_A 位，如果为 1 则记录在 mask 中，随后将 PTE_A 手动清 0。最后，通过 copyout 将结果拷贝给用户即可，函数代码如下：

   ```c
   // kernel/sysproc.c
   int
   sys_pgaccess(void)
   {
     // lab pgtbl: your code here.
     uint64 vaddr;
     int num;
     uint64 res_addr;
     argaddr(0, &vaddr);
     argint(1, &num);
     argaddr(2, &res_addr);
   
     struct proc *p = myproc();
     pagetable_t pagetable = p->pagetable;
     uint64 res = 0;
   
     for(int i = 0; i < num; i++){
       pte_t* pte = walk(pagetable, vaddr + PGSIZE * i, 1);
       if(*pte & PTE_A){
         *pte &= (~PTE_A);
         res |= (1L << i);
       }
     }
   
     copyout(pagetable, res_addr, (char*)&res, sizeof(uint64));
   
     return 0;
   }
   ```

   ![image-20240717160532286](.\images\image-20240717160532286.png)

   

   

## 实验中遇到的问题和解决方法

**问题**：在第一个和第三个实验中代码完成后总是有报错，显示未定义的函数调用

**解决方法**：不能忘记在`def.h`中加入函数声明

## 实验心得

在完成Lab: page tables的实验过程中，我深入理解了分页机制的实现细节和虚拟内存管理的重要性，通过动手实现和调试相关功能，我不仅掌握了如何操作页表，还增强了对操作系统底层原理的理解，这些实践经验对我今后进一步研究和开发操作系统有着重要的指导意义。

