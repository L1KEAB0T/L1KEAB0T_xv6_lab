# Lab2: system calls

## 实验目的

向xv6添加一些新的系统调用，这将帮助您了解它们是如何工作的，并使您了解xv6内核的一些内部结构。

## 实验过程

首先，创建一个syscall分支并且转移到此分支

```bash
git checkout -b syscall
```

### System call tracing

**要求**：需要添加一个系统调用跟踪功能、

1. 将`$U/_trace`添加到UPROGS中

2. 定义 trace 系统调用的原型, entry 和系统调用号

   在 `user/user.h`中添加 trace 系统调用原型

   ```c
   int trace(int);	// lab 2.1
   ```

   在 `user/usys.pl` 脚本中添加 trace 对应的 entry

   ```c
   entry("trace");	# lab 2.1
   ```

   在 `kernel/syscall.h` 中添加 trace 的系统调用号

   ```c
   #define SYS_trace 22	// lab 2.1
   ```

3. 在文件开头给内核态的系统调用 `trace` 加上声明，在 **kernel/syscall.c** 加上：

   ```c
   extern uint64 sys_trace(void);
   ```

   把新增的 `trace` 系统调用添加到函数指针数组 `*syscalls[]` 上：

   ```c
   static uint64 (*syscalls[])(void) = {
     ...
     [SYS_trace]   sys_trace,
   };
   ```

4.  `trace` 系统调用应该有一个参数，一个整数“mask(掩码)”，其指定要跟踪的系统调用。所以，我们在 **kernel/proc.h** 文件的 `proc` 结构体中，新添加一个变量 `mask` ，使得每一个进程都有自己的 `mask` ，即要跟踪的系统调用。

   ```c
   struct proc {
     ...
     int mask;               // Mask
   };
   ```

5. 然后就可以在 **kernel/sysproc.c** 给出 `sys_trace` 函数的具体实现了，只要把传进来的参数给到现有进程的 `mask` 就好了：

   ```c
   uint64
   sys_trace(void)
   {
     int mask;
     // 取 a0 寄存器中的值返回给 mask
     if(argint(0, &mask) < 0)
       return -1;
     
     // 把 mask 传给现有进程的 mask
     myproc()->mask = mask;
     return 0;
   }
   ```

6. 接下来就要把输出功能实现因为 `proc` 结构体(见 **kernel/proc.h** )里的 `name` 是整个线程的名字，不是函数调用的函数名称，所以我们不能用 `p->name` ，而要自己定义一个数组，我这里直接在 **kernel/syscall.c** 中定义了，这里注意系统调用名字一定要按顺序，第一个为空，当然你也可以去掉第一个空字符串，但要记得取值的时候索引要减一，因为这里的系统调用号是从 `1` 开始的。

   ```c
   static char *syscall_names[] = {
     "", "fork", "exit", "wait", "pipe", 
     "read", "kill", "exec", "fstat", "chdir", 
     "dup", "getpid", "sbrk", "sleep", "uptime", 
     "open", "write", "mknod", "unlink", "link", 
     "mkdir", "close", "trace"};
   123456
   ```

   然后我们就可以在 **kernel/syscall.c** 中的 `syscall` 函数中添加打印调用情况语句。 `mask` 是按位判断的，所以判断使用的是按位运算。进程序号直接通过 `p->pid` 就可以取到，函数名称需要从我们刚刚定义的数组中获取，即 `syscall_names[num]` ，其中 `num` 是从寄存器 `a7` 中读取的系统调用号，系统调用的返回值就是寄存器 `a0` 的值了，直接通过 `p->trapframe->a0` 语句获取即可。注意上面说的那个空格。

   ```c
   void
   syscall(void)
   {
     int num;
     struct proc *p = myproc();
   
     num = p->trapframe->a7;
     if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
       p->trapframe->a0 = syscalls[num]();
       // 下面是添加的部分
       if((1 << num) & p->mask) {
         printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num], p->trapframe->a0);
       }
     } else {
       printf("%d %s: unknown sys call %d\n",
               p->pid, p->name, num);
       p->trapframe->a0 = -1;
     }
   ```

7. 修改 `fork()` 函数（参见`kernel/proc.c`），将父进程的跟踪掩码复制到子进程。

   ![image-20240716164415603](.\images\image-20240716164415603.png![image-20240716184759680](D:\大学学习\大二下\操作系统小学期\labs_reports\images\image-20240716184759680.png)

   ## Sysinfo

   **要求**：添加一个系统调用 `sysinfo` ，它收集有关正在运行的系统信息。

   1. 将 `$U/_sysinfotest` 添加到 `Makefile` 的 `UPROGS` 中

   2. 在 **user/user.h** 中声明 `sysinfo()` 的原型，您需要预先声明 `struct sysinfo` ：

      ```C
      int sysinfo(struct sysinfo *);
      ```

      在 **kernel/syscall.h** 添加宏定义 `SYS_sysinfo` 如下：

      ```c
      #define SYS_sysinfo  23
      ```

      在 **user/usys.pl** 文件加入下面的语句：

      ```pl
      entry("sysinfo");
      ```

      然后在 **user/user.h** 中添加 `sysinfo` 结构体以及 `sysinfo` 函数的声明：

      ```c
      struct stat;
      struct rtcdate;
      // 添加 sysinfo 结构体
      struct sysinfo;
      
      // system calls
      ...
      int sysinfo(struct sysinfo *);
      ```

      在 **kernel/syscall.c** 中新增 `sys_sysinfo` 函数的定义：

      ```c
      extern uint64 sys_sysinfo(void);
      ```

      在 **kernel/syscall.c** 中函数指针数组新增 `sys_trace` ：

      ```c
      [SYS_sysinfo]   sys_sysinfo,
      ```

      记得在 **kernel/syscall.c** 中的 `syscall_names` 新增一个 `sys_trace` ：

      ```c
      static char *syscall_names[] = {
        "", "fork", "exit", "wait", "pipe", 
        "read", "kill", "exec", "fstat", "chdir", 
        "dup", "getpid", "sbrk", "sleep", "uptime", 
        "open", "write", "mknod", "unlink", "link", 
        "mkdir", "close", "trace", "sysinfo"};
      ```

   3. 写获取可用进程数目的函数实现，阅读 **kernel/proc.c** 文件可以找到数组proc保存了所有的进程。进程里面已经保存了当前进程的状态，所以我们可以直接遍历所有进程，获取其状态判断当前进程的状态是不是为 `UNUSED` 并统计数目就行了。

      在 **kernel/proc.c** 中新增函数 `nproc` 如下，通过该函数以获取可用进程数目：

      ```c
      // Return the number of processes whose state is not UNUSED
      uint64
      nproc(void)
      {
        struct proc *p;
        // counting the number of processes
        uint64 num = 0;
        // traverse all processes
        for (p = proc; p < &proc[NPROC]; p++)
        {
          // add lock
          acquire(&p->lock);
          // if the processes's state is not UNUSED
          if (p->state != UNUSED)
          {
            // the num add one
            num++;
          }
          // release lock
          release(&p->lock);
        }
        return num;
      }
      ```

   4. 接下来实现获取空闲内存数量的函数。可用空间的判断在 **kernel/kalloc.c** 文件，定义了一个链表，每个链表都指向上一个可用空间，这里的 `kmem` 就是一个保存最后链表的变量。

      从 `end (内核后的第一个地址)` 到 `PHYSTOP (KERNBASE + 128*1024*1024)` 之间的物理空间以 `PGSIZE` 为单位全部初始化为 `1` ，然后每次初始化一个 `PGSIZE` 就把这个页挂在了 `kmem.freelist` 上，所以 `kmem.freelist` 永远指向最后一个可用页，那我们只要顺着这个链表往前走，直到 `NULL` 为止。所以我们就可以在 **kernel/kalloc.c** 中新增函数 `free_mem` ，以获取空闲内存数量：

      ```c
      // Return the number of bytes of free memory
      uint64
      free_mem(void)
      {
        struct run *r;
        // counting the number of free page
        uint64 num = 0;
        // add lock
        acquire(&kmem.lock);
        // r points to freelist
        r = kmem.freelist;
        // while r not null
        while (r)
        {
          // the num add one
          num++;
          // r points to the next
          r = r->next;
        }
        // release lock
        release(&kmem.lock);
        // page multiplicated 4096-byte page
        return num * PGSIZE;
      }
      ```

   5. 在 **kernel/defs.h** 中添加上述两个新增函数的声明：

      ```
      // kalloc.c
      ...
      uint64          free_mem(void);
      
      // proc.c
      ...
      uint64          nproc(void);
      ```

   6. 按照实验提示，添加 `sys_sysinfo` 函数的具体实现，这里提到 `sysinfo` 需要复制一个 `struct sysinfo` 返回用户空间。

      在 **kernel/sysproc.c** 文件中添加 `sys_sysinfo` 函数的具体实现如下：

      ```c
      // add header
      #include "sysinfo.h"
      
      uint64
      sys_sysinfo(void)
      {
        // addr is a user virtual address, pointing to a struct sysinfo
        uint64 addr;
        struct sysinfo info;
        struct proc *p = myproc();
        
        if (argaddr(0, &addr) < 0)
      	  return -1;
        // get the number of bytes of free memory
        info.freemem = free_mem();
        // get the number of processes whose state is not UNUSED
        info.nproc = nproc();
      
        if (copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
          return -1;
        
        return 0;
      }
      ```

      最后在 **user** 目录下添加一个 **sysinfo.c** 用户程序：

      ```c
      #include "kernel/param.h"
      #include "kernel/types.h"
      #include "kernel/sysinfo.h"
      #include "user/user.h"
      
      int
      main(int argc, char *argv[])
      {
          // param error
          if (argc != 1)
          {
              fprintf(2, "Usage: %s need not param\n", argv[0]);
              exit(1);
          }
      
          struct sysinfo info;
          sysinfo(&info);
          // print the sysinfo
          printf("free space: %d\nused process: %d\n", info.freemem, info.nproc);
          exit(0);
      }
      ```

   7. 在 **Makefile** 的 `UPROGS` 中添加：

      ```Makefile
      $U/_sysinfotest\
      $U/_sysinfo\
      ```

      ![image-20240716193131155](.\images\image-20240716193131155.png)

## 实验中遇到的问题以及解决办法

**问题：** 可能会遇到编译失败或者无法正确运行用户级程序的情况。

**解决方法：**

- 确保在修改内核代码后进行 `make clean`，以清除旧的编译文件，然后再执行 `make qemu`。

**问题**：第一次clone仓库时未将所有的分支pull下来，导致无法切换分支实验失败

**解决方法：**

- 重新clone仓库并正确pull所有分支。

## 实验心得

在进行xv6系统调用实验的过程中，我学到了如何在操作系统内核中添加和管理系统调用。通过阅读xv6手册的相关章节，我理解了系统调用的工作原理，并掌握了如何在用户态和内核态之间传递数据。实际操作中，我添加了新的系统调用，修改了内核代码以处理这些调用，并调试了相关问题。这些经验让我对操作系统内核的结构和功能有了更深刻的理解，提高了我在底层编程和系统开发方面的能力。