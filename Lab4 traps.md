## Lab4 traps

### 1. 理解栈和栈帧

故事要从一张图片说起：

![image-20220419232944626](/home/chen/.config/Typora/typora-user-images/image-20220419232944626.png)

进程的创建和程序的运行：

举个例子来说，比如shell要运行一个程序，首先通过fork来创建进程，allocproc会映射user address space顶部的trampoline和trapframe（用于处理trap），然后exec把可执行文件载入到text和data中，然后分配stack和gurde page。<font color=red>(注：写了lab5 lazy allocation以后，可以发现在fork和exec之间还需要sbrk来申请空间)</font>

+ **以下只是我的猜想，不一定准确**

  指令的信息都是放在text中的，理由：在user目录中，打开任何一个应用程序的.asm文件，可以发现每个函数的指令的地址都特别小，接近与0，说明它们都在虚拟地址的低处，查看上图可知，应该是在text段。<font color=red>（注：这已经被证明是正确的）</font>

+ **如何理解指令**

  内存中不光有数据，也有指令。指令也是在内存中的单元，它们也有自己的地址，由pc指向并执行。

+ **如何理解栈**

  ![img](https://github.com/huihongxiao/MIT6.S081/raw/master/.gitbook/assets/image%20%28263%29.png)

  我现在也不是很懂，暂且记下，方便以后修改和补充。<font color=red>**栈是程序的载体,，体现在变量的空间在栈上分配以及函数的调用需要栈**</font>。从c语言程序的角度来看，程序执行伴随着函数调用，函数的调用就需要保存现场和恢复现场，这个过程中就需要栈。当一个函数a调用函数b时，需要将函数a的各个状态都记下，以便从b返回时恢复。这些被记录下来的信息就放在栈空间（就是第一幅图的**stack**），主要保存return addre，to prev. frame，saved register，local variables，图中return address就在ra寄存器里。通过阅读汇编代码可以知道，当a调用b时，在去b之前，会先把这些信息存好，再跳转到b（主要是通过<font color=red>**jalr**</font>指令，跳转并链接寄存器，该指令会把PC+4存到ra，然后跳转到指定**地址**。不过也可以手动完成这一个步骤：先通过sd指令保存ra，在通过call指令到指定**函数**），而在b的开始，会先把sp（stack pointer）向下移动（<font color=red>要做减法，因为栈是向下生长的</font>），腾出空间，保存返回地址，再做相应的操作；若函数b要返回，则先将sp复原，通过**ret**指令返回。这就形成了<font color=red>**栈帧**</font>。**（注意，由于我也没学过汇编语言，以上都是不完全准确的，但是大概是这样）**
  
+ 一个例子：

  ```assembly
  int a = 1;
  14:	4785                	li	a5,1
  16:	fcf42623          	sw	a5,-52(s0)
  printf("%p\n", &a);	
  1a:	fcc40593          	addi	a1,s0,-52
  1e:	00001517          	auipc	a0,0x1
  22:	89a50513          	addi	a0,a0,-1894 # 8b8 <statistics+0x88>
  26:	00000097          	auipc	ra,0x0
  2a:	668080e7          	jalr	1640(ra) # 68e <printf>
  ```

  我在`echo.c`的main函数中加了`int a = 1`和`printf("%p\n", &a)`，即声明一个变量并打印它的地址，通过gdb调试如下：

  ![image-20220423205720693](/home/chen/.config/Typora/typora-user-images/image-20220423205720693.png)

  在fork，exec以及sys_sbrk上设置断点，continue，可以发现：

  ![image-20220423205840228](/home/chen/.config/Typora/typora-user-images/image-20220423205840228.png)

  载入并执行初始程序`init`，继续continue

  ![image-20220423210016937](/home/chen/.config/Typora/typora-user-images/image-20220423210016937.png)

  先fork，在载入并执行`shell`，此时我们可以进行交互了，输入echo hi后，继续continue

  ![image-20220423210307277](/home/chen/.config/Typora/typora-user-images/image-20220423210307277.png)

  中间多了一步sys_sbrk，用于扩大空间，然后载入并执行echo，此时在0x1a处设置断点（使PC停留在执行完`int a = 1;`i以后），continue。此时我们先来预测一下结果，看汇编代码，`li	a5,1`表示a5存的是1,也就是变量a的值，`sw	a5,-52(s0)`表示将a5的值存到s0-52的地址处，也就是a的地址，所以s0-52就表示变量a的内存单元，我们不妨打印这个地址和地址中的值：

  ![image-20220423211139617](/home/chen/.config/Typora/typora-user-images/image-20220423211139617.png)

  可以看到地址中的值正是1,而且值得注意的是变量a的地址在user address space中为`0x2f8c`，这个地址应该在第3页，从第一副图可以看出这个地址在guard page，<font color=red>**很奇怪**</font>，guard page中的地址都是非法的呀！原来在xv6 book中有如下论述：**xv6 programs have only one program section header, but other systems might have separate sections for instructions and data**.在xv6中text和data是合一起的，所以这个地址应该在stack中，而不是guard page中。

+ <font color=red>**一个疑问**</font>：在上述程序中，如果我不打印变量a的地址，而只是打印a的值，生成的汇编代码为：

  ```assembly
  int a = 1;
  printf("%d\n", a);
  14:	4585                	li	a1,1
  16:	00001517          	auipc	a0,0x1
  1a:	89a50513          	addi	a0,a0,-1894 # 8b0 <statistics+0x88>
  1e:	00000097          	auipc	ra,0x0
  22:	668080e7          	jalr	1640(ra) # 686 <printf>
  ```

  可以看到，`int a = 1;`这一句根本就**没有给任何寄存器赋值，也没有把任何值store进任何地址，而是直接略过了！**而是直接将1作为`printf`的第二个参数load进了a1当中。我猜测这是编译器的某种优化，因为在这种情况下，以上操作都是多余的，只会拖慢速度，占用空间。

### 2. RISC-V assembly(<font color=green>easy</font>)

该部分比较简单，就不讲了，不过里面的文章和资料还没有看，等看完以后在<font color=blue>此处补上笔记</font>

#### 2.1 分析

下面是riscv中的寄存器（不包含浮点数寄存器）

| ABI name | Description | saver |
| --- | --- | --- |
| zero | Hard-wired zero |  |
| ra | Return Address | Caller |
| sp | Stack Pointer | Callee |
| gp | Global Pointer |  |
| tp | Thread Pointer |  |
| t0-2 | Temporaties | Caller |
| s0/fp | Saved Register/Frame Pointer | Callee |
| s1 | Saved Register | Callee |
| a0-1 | Fanction Arguments/Return Value | Caller |
| a2-7 | Function Arguments | Caller |
| s2-11 | Saved Register | Callee |
| t3-6 | Temporaries | Caller |



### 3. Backtrace(<font color=blue>moderate</font>)

#### 3.1 题目

在`kernel/printf.c`中完成一个backtrace函数，在sys_sleep中调用该函数，然后运行buttest，buttest会调用sys_sleep。编译器为每个stack frame放一个fp，这个fp存着调用该函数的fp的地址（即fp指向父函数的fp），backtrace沿着这一串链来打印每一帧的返回地址ra。

提供了函数

```c
static inline uint64
r_fp()
{
  uint64 x;
  asm volatile("mv %0, s0" : "=r" (x) );
  return x;
}
```

来获取当前栈帧的fp

#### 3.2 分析

我们分析一下整个过程，运行buttest，buttest调用sys_sleep，最后sys_sleep调用backtrace，此时应该至少有三个stack frame，buttest在最顶端，backtrace在最底端。<font color=red>**需要注意的是，根据提示，return address和to prev. fram在stack frame中具有固定的位置，即return address一定是在fp的-8位置，而to prev. frame则一定是在fp的-16位置，根据这个，我们就可以访问到所有stack frame的ra了**</font>

#### 3.3 实现

```c
void
backtrace(void)
{
  printf("backtrace:\n");
  uint64 fp = r_fp();
  uint64 botton = PGROUNDDOWN(fp);
  uint64 top = PGROUNDUP(fp);
  while (fp > botton && fp < top) {
    printf("%p\n", *(uint64 *)(fp - 8));
    if (fp - 16 < botton)
      break;
    fp = *(uint64 *)(fp - 16);
  }
}
```

循环终止的条件：stack的空间是一页，通过对fp上取一页，下取一页，得到fp的范围。



### 4. Alarm(<font color=red>hard</font>)

#### 4.1 题目

上面所有讲到的东西似乎和trap一点关系都没有。。。不过这题就有关系了。

这个实验要求我们在xv6中增加1项特性：在一个进程使用cpu时定期警报。这对与那些希望限制CPU时间消耗的受计算限制的进程，或者对于那些计算的同时执行某些周期性操作的进程可能很有用。

需要添加一个新的syscall：`sigalarm(interval, handler)`，其中intercal是时间间隔，即每interval就警报一次，handler是警报的具体内容，比如在终端中打印一段话。<font color=red>**如果一个程序调用了`sigalarm(n, fn)`那么，每隔n个cpu时间（ticks），kernel就要调用fn来提示警报。当fn返回时，程序需要返回到它离开的位置（因为被中断了）继续执行**</font>

#### 4.2 分析

关于trap的原理，这里不多说。具体可以看课程视频。

一个进程trap，可能有3种情况，第一是syscall，因为若要进行系统调用，必须从user space到kernle space；第二是device interrupt（设备中断），比如磁盘完成了读和写都会触发中断，此外还有**计时器的中断，也就是这个实验涉及到的中断**，它每个一段时间就会触发；第三是exception（异常），比如说使用了非法的虚拟地址如**page fault**，以及除0等操作都会触发异常。

~~众所周知，一个进程是不会自己报警的（这不是废话么。。。）~~所以我们需要打开一个**开关**，来让进程拥有此功能，而承担开关功能的就是`sigalarm`函数，请看提供的alarmtest.c的部分代码：

```c
void
test0()
{
  int i;
  printf("test0 start\n");
  count = 0;
  sigalarm(2, periodic);
  for(i = 0; i < 1000*500000; i++){
    if((i % 1000000) == 0)
      write(2, ".", 1);
    if(count > 0)
      break;
  }
    //后面的省略
}
```

函数`sigalarm`当然**不可能**阻塞在那里，然后等着时间流逝，然后发出一个警报，这样实在是太愚蠢了。肯定是`sigalarm`调用时，<font color=red>**通过某种方式，修改了进程的一些信息**</font>，调用完后，马上执行后续的代码，可以看出这里的后续代码就是一个很大很大的循环，看到这里，实际上这个lab的意思已经很清楚了！由于这个循环会占用大量的时间，**每隔一定的时间，计时器都会触发中断，进入内核，就下来就是在`usertrap`中处理这个中断了！**

由于一个进程的信息都在`struct proc`中，所以我们需要在其中加入一些必要的信息，比如两次报警之间的间隔（即interval），以及<font color=red>**句柄**</font>（即要执行的报警函数hendler）。

#### 4.3 第一次实现

在`proc.h`中的结构体`struct proc`中加入信息：

```c
struct proc {	
  //增加的信息
  uint64 fn;                 // the address of periodic
  int n;                     // tick num
  int history;               // ticks done
}
```

在`sys_sigalarm`中获取两个参数并扔进proc中

```c
uint64
sys_sigalarm(void)
{
  int n;
  uint64 fn;
	
  if (argint(0, &n) < 0)
    return -1;
  if (argaddr(1, &fn) < 0)
    return -1; 
  myproc()->fn = fn;	
  myproc()->n = n;
  myproc()->history = 0;

  return 0;
}
```

在usertrap中对计时器中断进行处理：

```c
void
usertrap(void)
{
  //...
  if(which_dev == 2) {
  if (++p->history == p->n) {
      p->history = 0;
      p->busy = 1;
      p->trapframe->epc = p->fn;
    }
  }
  //...
}
```

中断之前的信息都保存在trapframe中了，这里修改了trapframe-epc，**目的是让userret不要回到被中断的地方继续执行，而是去handler指向的位置执行报警函数**。<font color=blue>这里留个疑问，为什么不能直接调用handle，通过函数指针的方式来调用，虽然当前是在kernel space，kernel pagetable没有从handler的虚拟地址到物理地址的映射，但是我们可以通过p->pagetable来找到user space的pagetable，然后通过walkaddr找到物理地址。事实上我一开始就是采用这种方式，但是没有成功。我想我可能是忘记了mappages这va和pa了，因为在mmu打开的状态下，一切地址都被认为是va，硬件自动walk来找到pa，虽然kernel address是恒等映射，但是我甚至没有映射，可能这里出错。</font><font color=red>已破案，感谢群友的解答。因为handler是用户态的函数，如果这个函数在内核态调用，那就运行在内核栈上，内核态有特权，如果用户程序是恶意的，那就存在很大的安全隐患，此外，如果函数内调用用户态的全局变量或者其他函数，有需要手段翻译（因为内核栈上没有这些信息），会很麻烦。</font>

#### 4.4 再次分析

如果按照这样思路，那么通过test0没有问题，但是无法通过test1。因为我们忘记考虑了一个很重要的因素：恢复现场

我们来梳理一下整个过程，用户进程被中断以后，通过trampoline进入usertrap，假设此时trapframe中的状态是`状态a`，该状态是离开userspace时的状态，从kernel中返回时需要载入该状态。在usertrap中，我们修改了trapframe->epc，为的是返回user space时不回到被中断的地方，而是进入handler，此时trapframe中的状态还是`状态a`。而在handler中，有一个`printf("alarm!")`,这里是需要**系统调用**的，同要要通过trampoline进入usertrap，<font color=red>**因为trampoline要保存现场，它会把原有的现场覆盖掉，此时trapframe中的状态已经不是`状态a`了，而是`状态b`！**</font>因而最后返回被中断位置的时候，恢复的状态不是初始状态。test0可以通过的原因是，从它并没有对此进行测试，换句话说，只要顺着这条线走，返回到被中断位置就可以通过，而test1对此进行了测试。

test2没啥好说的，一个中断处理正忙时，不能处理下一个中断。

#### 4.5 再次实现

+ 在`proc.h`加入新的变量`struct trapframe add_trapframe`用于暂存状态a，以及busy变量来表示trap是否正忙

+ 在usertrap中将p->trapframe拷贝到p->add_trapframe

+ 在sigreturn中将p->add_trapframe拷贝回p->trapframe，**不得不说，sigreturn的存在真是一个绝妙的设计**

```c
void
usertrap(void)
{
  //...
  if(which_dev == 2) {
    if (p->busy == 0 && ++p->history == p->n) {
      memmove(&(p->add_trapframe), p->trapframe, sizeof(p->add_trapframe));
      p->history = 0;
      p->busy = 1;
      p->trapframe->epc = p->fn;
    }
  }
  //...
}
```

```c
uint64
sys_sigreturn(void)
{
  memmove(myproc()->trapframe, &(myproc()->add_trapframe), sizeof(myproc()->add_trapframe));
  myproc()->busy = 0;
  return 0;
}
```

