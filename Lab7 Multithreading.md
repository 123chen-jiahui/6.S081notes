## Lab7 Multithreading
~~迄今为止做到的最简单的lab~~


### 1. Uthread: switching between threads(<font color = blue>moderate</font>)

#### 1.1 题目
可以算是对课程里讲的线程切换的简化实现版本。已知给了`user/uthread.c`和`user/uthread_switch.S`，`uthread.c`中有三个线程，`thread_a`和`thread_b`以及`thread_c`，实现相关的代码，使得能够这三个线程能够切换。
<br>

#### 1.2 分析
个人认为，这道题目虽然不难，但是这为我们自己实现线程库具有十分重要的意义。我们总是用到线程库`pthread`，也总是使用里面的一些数据类型，比如说`pthread_t`，以及线程库中的函数，比如说`pthread_create`，线程库会帮助我们自动切换线程，但是我们对这一过程并不熟悉。学了课程以后再来做这个实验，一定程度上启发了我们如何自己实现一个简易的线程库。

下面是`uthread.c`的源代码分析：

1. 结构体和全局变量：
    ```c
    struct thread {
      char stack[STACK_SIZE]; /* the thread's stack */
      int state;              /* FREE, RUNNING, RUNNABLE */
      struct context context; /* save regs */
    };
    ```
    可以说，这在一定程度上实现了`pthread_t`，里面两个成员表示线程运行所需的栈以及线程运行的状态。可以猜到，当线程切换的时候，切换线程和被切换线程的运行状态一定会改变。

    `all_thread`用于管理所有进程，`current_thread`用于表示当前进程，这些都是必不可少的，`context`是为了完成寄存器状态保存加的。

2. `thread_init`函数，这个函数十分重要，它是整个进程的第0个线程（也就是main函数），从这个线程开始调度到了其他线程( `thread_a`,`thread_b`,`thread_c`)，并且调度器再也不会切换回到第0个进程，因为它的state被设为了`RUNNING`，并且不再改变.

3. `thread_create`:
    ```c
    void
    thread_create(void (*func)())
    {
      struct thread *t;
    
      for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
        if (t->state == FREE) break;
      }
      t->state = RUNNABLE;
      // YOUR CODE HERE
      t->context.ra = (uint64)func;
      t->context.sp = (uint64)t->stack + PGSIZE;
    }
    ```
    线程的创建，指定线程要运行的函数，并为其分配空余的线程编号(即`t->state==FREE`)。值得注意的是，不同于Linux的`pthread_create`，它会直接开始运行此线程，但是这里只是将线程状态改为`RUNNABLE`，并没有运行，真正运行是在调度器`thread_schedule`中。<font color = red>等一下其他线程通过`uthread_switch`切换到此线程，但是这个线程刚刚创建，并没有调用过switch，所以需要“伪造”一个对switch的调用，即：将ra设置为要执行函数的第一条指令的地址(在这里就是func)，同时将栈设置好。</font>

4.  yield和调度器

    这两个函数基本上就是kernel中的`yield`和`schedule`的简易版本，在`schedule`中加上`switch`即可，而`switch`可以完全仿照kernel中的来写。

#### 1.3 思考

+ 可以注意到没有用到锁。为什么，kernel中的实现用到了锁。个人感觉是因为这里的并发，可能不是真正的并发，在调度器中，是完全按照顺序来切换线程的，除了第0个线程以外，在一个时间内，三个线程中只有1个会在执行，而且不会有中断。因而不存在race condition，所以不需要加锁。
+ 如何理解每个线程的stack？我们知道，程序的运行是需要栈的（因为存在函数调用，以及存放局部变量、函数参数等），每个线程运行在不同的栈上，所以要为每个线程在`thread_create`时设置栈指针sp。在上面的代码中，设置了栈的大小为PGSIZE(4096B)，实际上经测试，不需要这么大，1024B也行。我觉得这与**函数调用深度**有关。一个反复递归的函数必然占据大量的栈空间。

### 2. Using threads(<font color = blue>moderajte</font>)

这题感觉没啥好讲的。引起race condition的原因在lock那节课上讲过：因为链表的插入方法采用头插法，问题出在当插入新的结点时，链表的头结点是否选对了，所以导致可能会有迷途的结点。

answer-thread.txt：我们检查的key，而不是value，所以对于任何一条线程来说，它们访问到的结点总数是一样的，所以两个线程都丢失了键

改进后的put

```c
static
void put(int key, int value)
{
  int i = key % NBUCKET;

  // is the key already present?
  struct entry *e = 0;

  pthread_mutex_lock(&mutex[i]);
  for (e = table[i]; e != 0; e = e->next) {
    if (e->key == key)
      break;
  }
  if(e){
    // update the existing key.
    e->value = value;
  } else {
    // the new is new.
    insert(key, value, &table[i], table[i]);
  }
  pthread_mutex_unlock(&mutex[i]);
}
```

有一个问题：当线程数很多时，比如说10000个线程，程序执行效率会变的极低。



### 3. Barrier(<font color = blue>moderate</font>)

这题也没啥好讲的。但是有一点，我不确定，但我认为这是`coordination`的一种实现方式，课程里详细介绍的是sleep&wakeup，但是层提到过用`semaphore`（信号量）实现coordination。也许在vx6book中有，我没有仔细看。

代码如下：

```c
static void
barrier()
{
  // YOUR CODE HERE
  //
  // Block until all threads have called barrier() and
  // then increment bstate.round.
  //
  pthread_mutex_lock(&bstate.barrier_mutex);
  if (++bstate.nthread != nthread)
    pthread_cond_wait(&bstate.barrier_cond, &bstate.barrier_mutex);
    else {
      bstate.round ++;
      bstate.nthread = 0;
      pthread_cond_broadcast(&bstate.barrier_cond);
    }
    pthread_mutex_unlock(&bstate.barrier_mutex);
}
```

