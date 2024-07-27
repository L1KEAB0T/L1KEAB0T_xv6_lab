# Lab4: traps

## 实验目的

本实验探索如何使用陷阱实现系统调用

## 实验过程

### RISC-V assembly

1.执行`make fs.img`编译文件user/call.c

2.阅读***call.asm\***中函数`g`、`f`和`main`的代码

3.回答以下问题

- Q1: Which registers contain arguments to functions? For example, which register holds `13` in main’s call to `printf`?
  A: 函数参数的寄存器为 a0~a7. `printf` 的 `13` 存在寄存器 a2 中

- Q2: Where is the call to function `f` in the assembly code for main? Where is the call to `g`? (Hint: the compiler may inline functions.)
  A: 在 40 行可以看到, 编译器进行了函数内联, 直接将 `f(8)+1`的值 `12` 计算出来了.

- Q3: At what address is the function `printf` located?
  A: 由第 43 和 44 行可以看出, `jalr` 跳转的地址为 `0x30+1536=0x630`, 即函数 `printf` 的地址为 `0x630`

- Q4: What value is in the register `ra` just after the `jalr` to `printf` in `main`?
  A: 根据 `jalr` 指令的功能, 在刚跳转后 `ra` 的值为 `pc+4=0x34+4=0x38`.

- Q5: Run the following code.

  ```c
  unsigned int i = 0x00646c72;
  printf("H%x Wo%s", 57616, &i);
  ```

  What is the output?

  A: 输出: HE110 World![image-20240717164233198](.\images\image-20240717164233198.png)

  Q6: In the following code, what is going to be printed after `y=`? (note: the answer is not a specific value.) Why does this happen?

  ```c
  printf("x=%d y=%d", 3);
  ```

  A: 根据函数的传参规则, `y=` 后跟的值应该为寄存器 a2 的值.![image-20240717164406543](.\images\image-20240717164406543.png)

### Backtrace 

**要求**：编写函数 `backtrace()`, 遍历读取栈帧(frame pointer)并输出函数返回地址.

1. 在文件 `kernel/riscv.h` 中添加内联函数 `r_fp()` 读取栈帧值

   ```c
   // read the current frame pointer from s0 register - lab4-2
   static inline uint64 r_fp() {
       uint64 x;
       asm volatile("mv %0, s0" : "=r" (x) );
       return x;
   }
   ```

2. 在 `kernel/printf.c` 中编写函数 `backtrace()`输出所有栈帧
   初始通过调用上述的 `r_fp()` 函数读取寄存器 s0 中的当前函数栈帧 fp. 根据 RISC-V 的栈结构, fp-8 存放返回地址, fp-16 存放原栈帧. 进而通过原栈帧得到上一级栈结构, 直到获取到最初的栈结构.

   ```c
   // print the return address - lab4-2
   void backtrace() {
       uint64 fp = r_fp();    // 获取当前栈帧
       uint64 top = PGROUNDUP(fp);    // 获取用户栈最高地址
       uint64 bottom = PGROUNDDOWN(fp);    // 获取用户栈最低地址
       for (; 
           fp >= bottom && fp < top;     // 终止条件
           fp = *((uint64 *) (fp - 16))    // 获取下一栈帧
           ) {
           printf("%p\n", *((uint64 *) (fp - 8)));    // 输出当前栈中返回地址
       }
   }
   
   ```

3. 添加 `backtrace()` 函数原型到 `kernel/defs.h`

4. 在 `kernel/sysproc.c` 的 `sys_sleep()` 函数中添加对 `backtrace()` 的调用.

5. 在 `kernel/printf.c` 的 `panic()` 函数中添加对 `backtrace()` 的调用.

   ![image-20240717171508601](.\images\image-20240717171508601.png)

### Alarm

**要求**：在一些应用场景中，我们希望，当一些进程运行了一段时间后，os会发出警告。这对于一些我们想要限制cpu，或者想定期采取一些行动的进程特别有用。

1. 修改makefile，加入alarmtest

   ```makefile
   	$U/_alarmtest\
   ```

2. 在user/user.h里加声明，hints里已经给出

   ```
   // add new alarm
   int sigalarm(int ticks, void (*handler)());
   int sigreturn(void);
   ```

   接下来这部分与Lab2添加系统调用大同小异

   user/usys.pl增加

   ```
   entry("sigalarm");
   entry("sigreturn");
   ```

   kernel/syscall.h增加

   ```
   #define SYS_sigalarm 22
   #define SYS_sigreturn 23
   ```

   kernel/syscall.c

   ```
   extern uint64 sys_sigalarm(void);
   extern uint64 sys_sigreturn(void);
   
   static uint64 (*syscalls[])(void) = {
   [SYS_sigalarm] sys_sigalarm,
   [SYS_sigreturn] sys_sigreturn,
   };
   ```

3. 接下来，需要让`sys_sigalarm`记录时钟间隔，为此，先在kernel/proc.h的proc中增添属性：

   alarm_trapframe是为了test1/2中的临时保存寄存器。

   ```
     // add for alarm
     int alarm_interval;
     int alarm_ticks;
     uint64 alarm_handler;
     struct trapframe alarm_trapframe;
   };
   ```

   

   并在kernel/proc.c的allocproc中初始化这些属性

   ```
     // set alarm related parameters
     p->alarm_interval = 0;
     p->alarm_ticks = 0;
     p->alarm_handler = 0;
   ```

   

   在kernel/trap.c的usertrap里处理interrupt

   ```
     // give up the CPU if this is a timer interrupt.
     if(which_dev == 2){
       if (p->alarm_interval){
         if (++p->alarm_ticks == p->alarm_interval){
           memmove(&(p->alarm_trapframe), p->trapframe, sizeof(*(p->trapframe)));
           p->trapframe->epc = p->alarm_handler;
         }
       }
       yield();
     }
   ```

   

   最后在kernel/sysproc.c中完善相关函数

   ```
   uint64 sys_sigalarm(void){
     int interval;
     uint64 handler;
   
     if (argint(0, &interval) < 0)
       return -1;
     if (argaddr(1, &handler) < 0)
       return -1;
   
     myproc()->alarm_interval = interval;
     myproc()->alarm_handler = handler;
     return 0;
   }
   
   uint64 sys_sigreturn(void){
     memmove(myproc()->trapframe, &(myproc()->alarm_trapframe), sizeof(struct trapframe));
     myproc()->alarm_ticks = 0;
     return 0;
   }
   ```

## 实验中遇到的问题及解决方法

### trapframe结构体理解困难

**问题**：我发现trapframe结构体难以理解，它包括了多个关键寄存器的信息，但我不确定这些寄存器如何在trap处理过程中起作用。

**解决方法**：

- **查阅资料**：我查阅了xv6的文档和相关资料，了解到trapframe结构体用于保存陷阱发生时CPU的状态，包括通用寄存器、程序计数器、标志寄存器等。
- **代码注释**：通过阅读和注释`trap.c`和`syscall.c`中的代码，我逐步理解了trapframe结构体的字段与具体寄存器之间的对应关系。

## 实验心得

在进行Lab4：traps实验过程中，我深刻理解了陷阱处理机制的重要性，并通过解决各种问题提升了调试能力。面对trapframe结构体的困惑，我通过查阅资料和阅读代码逐步明晰了寄存器的作用，最终实现了稳定的系统调用和中断处理。这次实验不仅让我在操作系统开发方面获得了宝贵经验，也增强了我解决复杂问题的信心。