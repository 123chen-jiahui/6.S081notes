这里讲讲有关锁的**基础**知识点

1. coordination
   
   为啥需要coordination？当我们在写线程代码时，有些场景需要**等待一些特定的事件/条件**，或者不同的线程之间需要**交互**，考虑这样一种情形：有一个进程从pipe中读数据，需要等待一个pipe不空的事件，所以就有：
   
   ```c
   void busy_wait()
   {
    while (pipe is empty)
         ;
    // other things 
   }
   ```
   
   以上是**busy wait（忙等待）**，即占用CPU并一直在spin。这浪费了CPU资源，因为很可能过了很久pipe内都不会有内容，而这些时间我们本可以做其他有意义的事。所以，我们通过switch函数调用的方式让出CPU，直到等待的事件发生再恢复执行。这就是coordination。coordination的实现方式有很多种，xv6使用的是**sleep&wakeup**。c语言线程库pthread使用的是**semaphore（信号量）**，这一点在上一个lab介绍过了。

2. sleep和wakeup。
   
   看看我们期望sleep&wakeup的功能：
   
   ```c
   void func1() 
   {
     while (flag == 0) // flag == 1即表示等待的事件
         sleep(...);
     // other things
     flag = 0;
   }
   
   void func2()
   {
   
     flag = 1;
     wakeup(...);
     // other things
   }   
   ```
   
   awakeup需要知道哪个/哪些进程是需要它唤醒的，所以sleep和wakeup时需要指定一个“暗号”**channel**。所以sleep和wakeup需要一个64bit的参数channel。
   
   broken_sleep:
   
   ```c
   void sleep(void *chan)
   {
   p->state = SLEEPING;
   p->chan = chan;
   swtch();
   // other things
   }
   ```
   
   为了保护flag这个不变量，需要在func1和func2加锁，这把锁称为 **condition lock**如下：
   
   ```c
   void func1() 
   {
      acquire(&flag_lock);
      while (flag == 0) // flag == 1即表示等待的事件
          sleep(chan);
      // other things
      flag = 0;
      release(&flag_lock);
   }
   
   void func2()
   {
      acquire(&flag_lock);
      flag = 1;
      wakeup(chan);
      release(&flag_lock);
      // other things
   }
   ```
   
   这样会造成一个问题：sleep切换到了其他进程，但是却保留了原进程的flag_lock，同时它需要等wakeup来唤醒，而wakeup又需要获取flag_lock，于是造成了死锁。解决方法是在sleep之前解锁，sleep结束后重新加锁：
   
   ```c
   void func1() 
   {
     acquire(&flag_lock);
     while (flag == 0) { // flag == 1即表示等待的事件
         release(&flag_lock);        
         sleep(chan);
         acquire(&falg_lock);
     }
     // other things
     flag = 0;
     release(&flag_lock);
   }
   ```
   
   但是这样又有问题：wakeup可能发生在release和sleep之间，也就是说，wakeup看到了这样一种状态：锁被释放了，但是进程还没有进入到SLEEPING状态。需要消除这一段间隙，**sleep需要将释放锁和设置进程为SLEEPING状态这两个行为合并为一个原子操作**。怎么做呢，由于wakeup在唤醒某个进程之前，必须获取这个进程的锁，所以在sleep中可以先获取这个进程的锁，然后再释放condition lock，然后再将状态改为SLEEPING，最后switch。
   
   ```c
   void sleep(void *chan, struct spinlock *lk)
   {
     acquire(*p->lock);
     release(&lk);
     p->state = SLEEPING;
     p->chan = chan;
     swtch();
     // other things
   }
   ```
   
   规则：
   
   - 调用sleep时需要持有condition lock，这样sleep函数才能知道相应的锁
   
   - sleep函数只有在获取到进程的锁p->lock之后，才能释放condition lock
   
   - wakeup需要同时持有两个锁（一个是condition lock，另一个是进程锁p->lock）才能查看进程
