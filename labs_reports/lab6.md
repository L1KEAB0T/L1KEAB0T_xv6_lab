# Lab 6: Multi-**threading**

## 实验目的

体验一下多线程。

## 实验过程

### Uthread: switching between threads

**要求**：设计并实现一个用户级线程系统的上下文切换机制。补充完成一个用户级线程的创建和切换上下文的代码。

首先在user/uthread.c定义一个结构体，保存寄存器内容：

```c
struct thread_context{
  uint64 ra;
  uint64 sp;

  // callee-saved
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```

并加入到线程中

```c
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct     thread_context context; 
};
```

在`thread_schedule`中按提示加入thread_switch

```c
     * Invoke thread_switch to switch from t to next_thread:
     * thread_switch(??, ??);
     */
    thread_switch((uint64)&t->context, (uint64)&current_thread->context);
  } else
    next_thread = 0;
}
```

再修改`thread_create`，保存callee-save registers

```c
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  t->context.ra = (uint64)func;
  t->context.sp = (uint64)&t->stack[STACK_SIZE - 1];
}
```

最后修改`user/uthread_switch.S`，保存寄存器

```c
	.globl thread_switch
thread_switch:
	/* YOUR CODE HERE */
        sd ra, 0(a0)
        sd sp, 8(a0)
        sd s0, 16(a0)
        sd s1, 24(a0)
        sd s2, 32(a0)
        sd s3, 40(a0)
        sd s4, 48(a0)
        sd s5, 56(a0)
        sd s6, 64(a0)
        sd s7, 72(a0)
        sd s8, 80(a0)
        sd s9, 88(a0)
        sd s10, 96(a0)
        sd s11, 104(a0)

        ld ra, 0(a1)
        ld sp, 8(a1)
        ld s0, 16(a1)
        ld s1, 24(a1)
        ld s2, 32(a1)
        ld s3, 40(a1)
        ld s4, 48(a1)
        ld s5, 56(a1)
        ld s6, 64(a1)
        ld s7, 72(a1)
        ld s8, 80(a1)
        ld s9, 88(a1)
        ld s10, 96(a1)
        ld s11, 104(a1)
        ret    /* return to ra */
```

![image-20240718150304423](.\images\image-20240718150304423.png)

### Using threads

**要求**：通过使用线程和锁实现并行编程，以及在多线程环境下处理哈希表。

先在notxv6/ph.c声明

```c
pthread_mutex_t lock[NBUCKET];
```

再在main里初始化

```c
  // init lock
  for (int i = 0; i < NBUCKET; i++){
    pthread_mutex_init(&lock[i], NULL);
  } 
```

最后在put里上锁

```c
  } else {
    pthread_mutex_lock(&lock[i]);
    // the new is new.
    insert(key, value, &table[i], table[i]);
    pthread_mutex_unlock(&lock[i]);
  }
```

![image-20240718150206155](.\images\image-20240718150206155.png)

## Barrier

**要求**：通过实现一个线程屏障（barrier），即每个线程都要在 barrier 处等待，直到所有线程到达 barrier 之后才能继续运行，加深对多线程编程中同步和互斥机制的理解。

直接按提示来

```c
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  bstate.nthread++;
  if (bstate.nthread == nthread) {
    bstate.round++;
    bstate.nthread = 0;
    pthread_cond_broadcast(&bstate.barrier_cond);
  }
  else {
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
  }
  pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

![image-20240718151112334](.\images\image-20240718151112334.png)

## 实验中遇到的问题及解决方法

本次实验难度不大，特别是结果上学期写电梯调度后

## 实验心得

在Lab 7的Multithreading实验中，我深入理解了多线程编程的重要性和挑战。通过探索线程创建、同步和互斥操作，我学会了如何有效地利用多核处理器的并行能力，提升程序的性能和响应能力。这次实验不仅增强了我的编程技能，也加深了对操作系统并发机制的理解。