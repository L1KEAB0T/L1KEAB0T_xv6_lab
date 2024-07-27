# Lab 10: mmap

## 实验目的

这个实验需要在 xv6 中实现 `mmap` 功能。`mmap` 和 `munmap` 系统调用允许 UNIX 程序对其地址空间进行详细控制。 它们可用于在进程之间共享内存，将文件映射到进程地址空间，以及作为用户级页面错误方案的一部分。

## 实验过程

1. 在 `Makefile` 中添加 `$U/_mmaptest`

2. **添加系统调用：** 添加有关 `mmap` 和 `munmap` 系统调用的定义声明。包括 ：

   - `kernel/syscall.h`：

     ```
     #define SYS_mmap   22   // lab 10
     #define SYS_munmap 23   // lab 10
     ```

   - `kernel/syscall.c`：

   ```
   extern uint64 sys_mmap(void);   // lab 10
   extern uint64 sys_munmap(void); // lab 10
   
   static uint64 (*syscalls[])(void) = {
   ...
   [SYS_mmap]    sys_mmap,   // lab 10
   [SYS_munmap]  sys_munmap, // lab 10
   };
   ```

   - `user/usys.pl`：

     ```
     entry("mmap");      // lab 10
     entry("munmap");    // lab 10
     ```

   - `user/user.h`：

     ```
     void *mmap(void *, int, int, int, int, int);
     int munmap(void *, int);
     ```

3. **定义 `vm_area` 结构体及其数组：**

   1. 在 `kernel/proc.h` 中定义 `struct vm_area` 结构体。

      `vm_area` （VMA）用于表示使用 mmap 系统调用文件映射的虚拟内存的区域的位置、大小、权限等。结构体中具体记录的内存信息和 mmap 系统调用的参数基本对应, 包括映射的起始地址、映射内存长度(大小)、权限、mmap 标志位、文件偏移以及指向的文件结构体指针。

      ```
      struct vm_area {
          uint64 addr;    // mmap address
          int len;    // mmap memory length
          int prot;   // permission
          int flags;  // the mmap flags
          int offset; // the file offset
          struct file* f;     // pointer to the mapped file
      };
      ```

   2. 在 `struct proc` 结构体中定义相关字段。

      根据实验指导的提示，对于每个进程都使用一个 VMA 的数组来记录映射的内存。此处定义了 NVMA 表示 VMA 数组的大小，并在 struct proc 结构体中定义了 vma 数组，又因为 VMA 是进程的私有字段，因此对于 xv6 的进程单用户线程的系统，访问 VMA 无需加锁。

      ```
      #define NVMA 16
      struct proc {
          ...
          struct vm_area vma[NVMA];    // VMA array - lab10
      };
      ```

      

4. **定义权限和标志位：** 在 `kernel/fcntl.h` 中，已经定义了 `PROT_READ` 和 `PROT_WRITE` 权限。你需要为 `mmap` 系统调用实现 `MAP_SHARED` 和 `MAP_PRIVATE` 标志位。

5. **懒加载页表：** 类似于之前的惰性页分配实验，通过在页面错误处理代码中（或者被 `usertrap` 调用）懒加载页表。确保 `mmap` 不会分配物理内存或读取文件。

6. **记录虚拟内存区域：** 定义一个与 Lecture 15 中虚拟内存区域（VMA）相对应的结构，用于记录由 `mmap` 创建的虚拟内存范围的地址、长度、权限和文件等信息。在 xv6 内核中，可以声明一个固定大小的 VMA 数组，并根据需要从数组中分配。

7. **实现 `mmap`：** 找到进程地址空间中未使用的区域，用于映射文件，并将 VMA 添加到进程的映射区域表中。VMA 应包含指向要映射文件的 `struct file` 的指针。确保在 `mmap` 中增加文件的引用计数，以防止文件在关闭时被删除。

8. **处理页面错误：** 由于在 sys_mmap() 中对文件映射的内存采用的是 Lazy allocation，因此需要对访问文件映射内存产生的 page fault 进行处理。当在 `mmap` 映射的区域发生页面错误时，分配一个物理内存页，从相关文件中读取 4096 字节到该页面，并将其映射到用户地址空间。你可以使用 `readi` 函数从文件中读取数据。

9. **实现 `munmap`：** 找到要取消映射的地址范围的 VMA，并取消映射指定的页面。如果 `munmap` 移除了前一个 `mmap` 的所有页面，则应减少相应的 `struct file` 的引用计数。如果一个页面被修改且文件是 `MAP_SHARED` 映射，则将修改的页面写回文件。

   

10. **设置脏页标志位：**

    1. 在 `kernel/riscv.h` 中定义脏页标志位 `PTE_D`，表示页面已被修改。

       ```
       #define PTE_D (1L << 7) // dirty flag - lab10
       ```

    2. 在 `kernel/vm.c` 中实现 `uvmgetdirty()` 和 `uvmsetdirtywrite()` 两个函数，用于读取和设置脏页标志位。

    3. 在 `usertrap()` 函数中处理页面访问的 page fault。

    4. 对于未映射的可写页面进行写操作时，通过触发 page fault 实现 Lazy allocation。

    5. 在写操作触发的 page fault 处理中，根据 `r_scause()` 和 VMA 的权限设置，添加脏页标志位。

    6. 在 `munmap()` 函数中，通过调用 `uvmgetdirty()` 来确定页面是否被修改。

11. **修改`exit`和 `fork`的系统调用：** 上述内容完成了基本的 `mmap` 和 `munmap` 系统调用的部分，最后需要对 `exit` 和 `fork` 两个系统调用进行修改。

    1. 修改 `kernel/proc.c` 中的 `exit()` 函数。 在进程退出时，需要像 `munmap()` 一样对文件映射部分内存进行取消映射。因此添加的代码与 `munmap()` 中部分基本系统，区别在于需要遍历 VMA 数组对所有文件映射内存进行整个部分的映射取消。

    2. 修改 `kernel/proc.c` 中的 `fork()` 函数。 修改 `fork`，以确保子进程与父进程具有相同的映射区域。不要忘记增加 VMA 的 `struct file` 的引用计数。在子进程的页面错误处理程序中，可以分配新的物理页面，而不是与父进程共享页面。

       ![image-20240720111037932](.\images\image-20240720111037932.png)

## 实验遇到的问题和解决方案

**问题：** 调用`mmap`函数后返回`MAP_FAILED`。

**解决方案：** 检查传递的参数，确保文件描述符有效，映射的长度和偏移量合法。

## 实验心得

在`mmap`实验中，我学会了如何通过内存映射将文件内容直接映射到进程的地址空间，从而提高文件操作的效率。尽管遇到了一些如权限问题、内存泄漏等挑战，但通过调整参数和正确释放资源，我顺利解决了这些问题，增强了对内存管理和系统调用的理解。