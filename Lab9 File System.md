## Lab9 File System

由于本人这段时间忙于其他事务，这一篇可能会写得比较简略，并且文件系统的有关知识和前面课程的知识不同，文件系统的知识十分庞杂，相应的笔记都会记录在[另一篇文章](./fs.md)中。

### 1. Large files(<font color = blue>moderate</font>)

#### 1.1 题目

在原始xv6中，每个inode有13个`block number`（b0，b1，b2……），用来指定文件的数据所在的block，每个block number占4B。其中12个是"direct" block number，另一个是"singly-indirect" block number，这样xv6中一个文件最大可以是(256+12)\*1024B=268KB。为了能够表示更大的文件，可以引入"double-indirect" block number。加入有11个直接索引，1个间接索引，1个双重间接索引，这样一个文件最大可以是(11+256+256\*256)\*1024B=65803KB=64.3MB。

#### 1.2 实现

更改inode中block number的组成

```c
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))
#define NDOUBLE (NINDIRECT*NINDIRECT)
#define MAXFILE (NDIRECT + NINDIRECT + NDOUBLE)
```

根据提示，更改`static uint bmap(struct inode *ip, uint bn)`，该函数的作用是：对于**inode中**的第n个block，指出其在磁盘设备中的block num。下面是一个改进之前的inode block number示意图：

![](C:\Users\86151\Desktop\notes\6.S081notes\images\inode.jpg)

```c
static uint
bmap(struct inode *ip, uint bn)
{
    // ...
    if (bn < NDOUBLE) {
        // bmap 第一级目录
        if ((addr = ip->addrs[NDIRECT + 1]) == 0)
          ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
        bp = bread(ip->dev, addr);
        a = (uint*)bp->data;
        // bmap 第二级目录
        int num_single_block = bn / NINDIRECT;
        int left = bn % NINDIRECT;
        if ((addr = a[num_single_block]) == 0) {
          a[num_single_block] = addr = balloc(ip->dev);
          log_write(bp);
        }
        brelse(bp);
        bp = bread(ip->dev, addr);
        a = (uint*)bp->data;
        if ((addr = a[left]) == 0) {
          a[left] = addr = balloc(ip->dev);
          log_write(bp);
        }
        brelse(bp);
        return addr;
    }
    // ...
}
```

**注意每次bread某个block后，最后都要回收该block**

函数`void itrunc(struct inode *ip)`的作用是给定一个inode，释放该inode的所有空间，包括它索引数据区block。由于inode的索引结构有所变化，所以在回收时也要做出改变。原本itrunc中释放了一级索引和二级索引，现在需要释放三级索引。

```c
void
itrunc(struct inode *ip)
{
  // ...
  // 释放二级目录
  if (ip->addrs[NDIRECT + 1]) {
    struct buf *out_bp;
    uint *out_a;
    out_bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
    out_a = (uint*)out_bp->data;
    for (i = 0; i < NINDIRECT; i ++) {
      if (out_a[i]) {
        bp = bread(ip->dev, out_a[i]);
        a = (uint*)bp->data;
        for (j = 0; j < NINDIRECT; j ++) {
          if (a[j])
            bfree(ip->dev, a[j]);
        }
        brelse(bp);
        bfree(ip->dev, out_a[i]);
        // out_a[i] = 0;
      }
    }
    brelse(out_bp);
    bfree(ip->dev, ip->addrs[NDIRECT + 1]);
    ip->addrs[NDIRECT + 1] = 0;
  }
  // ...
}
```

这里需要注意的是，上述代码中`out_a[i] = 0;`这句是不需要的，所以被注释掉了，事实上，我们要清除的索引块号仅仅是在inode上的13个索引块号，包括二级索引和三级索引在内的索引块号是不用置0的（虽然置0也没关系），原因是这些用来存block num的块最终是要被回收的，在它们被再次分配时，balloc保证了将它们的所有字节置0.

### 2. Symlink Links

#### 2.1 题目

有关`symbolic link`（软链接/符号链接）的知识在xv6手册中似乎并没有介绍。`hard link`（硬链接）和`symlink`的知识我是在交大的银杏书《现代操作系统：原理与实现》中学习的，顺便复习了文件系统的有关知识（有一来说一，这本教材来很不错，很适合操作系统入门）。

这里简单用自己的语言讲讲这两个链接，就算是复习吧。

inode是个好东西，可以说是它组织了文件（这里的文件，说的是硬盘上存在的物理文件）。它记录了文件的`metadata`（元数据），通过inode，我们就可以访问文件中的数据。但是inode同时有个缺点：它是通过inumber为区分的，对于用户来说，这很不方便。于是我们引入了文件名来区分文件，而<font color = red>**管理文件名**</font>的是**目录**。目录也是文件，只不过它是`file with some datastructure`，这里的datastructure是一个称为`directory entry`（目录项）的东西，一个简单的目录项如下：

![](C:\Users\86151\Desktop\notes\6.S081notes\images\dirent.jpg)

可以这么理解，目录数据块存的数据就是一堆目录项，可以通过类似数组的方式访问。我们可以看出，文件名只是一个标识，它是依赖于inode的，信息真正存在于inode中。

Linux中硬链接命令：ln target path

完成的工作是创建一个新的文件path，它使用target的inode，即它俩共用一个inode，毋庸置疑，此操作后这个inode中的nlink=2。如果我们通过更改target中的内容来改变文件数据，那么显然这次更改对path来说是可见的。如果我们删除target，path并不会收到任何影响，只不过inode中的nlink变为了1。如果再将path删除，那么inode中的nlink变为0，操作系统会回收该inode。

Linux中软链接命令：ln -s target path

完成的工作是创建一个新文件path，<font color = red>**申请一个新的inode，文件的内容是target这个路径，也就是说该文件存了一个路径**</font>。这是一种特殊的文件——**符号链接文件**。如果尝试打开它，其实是打开了它存的路径中的那个文件，并且这个过程是递归的，如果路径中的文件仍是符号链接文件，继续跟随，直到不是符号链接文件。如果删除target，path就失去了了链接的对象，此时打开它会出错。

符号链接还可以链接目录，这个问题比较复杂，题目没有要求，这里不展开。

---

题目要求我们实现xv6的符号链接的系统调用。在xv6中，符号链接文件有两种打开方式：跟随或不跟随。所谓不跟随就是直接打开符号链接文件。

#### 2.2 实现

```c
uint64
symlink(char target[], char linkpath[])
{
  struct inode *ip;
  struct buf *bp;
  uint addr;
  ip = create(linkpath, T_SYMLINK, 0, 0); // 申请一个inode用来存新的符号链接文件.create中已经获得了锁
  // ilock(ip);
  if (ip == 0) {
    // iunlockput(ip);
    end_op();
    return -1;
  }
  // ilock(ip);
  if ((addr = ip->addrs[0]) == 0)
    ip->addrs[0] = addr = lazy_balloc(ip->dev);
  bp = bread(ip->dev, addr);
  memmove(bp->data, target, MAXPATH);
  iupdate(ip); // 将这个inode存盘
  iunlockput(ip);
  log_write(bp); // 将这个block存盘
  brelse(bp);

  end_op();
  return 0;
}
```

这里需要注意create中已经获得了锁，所以不能在外面再获得锁。还有最后解锁的时候要用iunlockput(ip)，它在解锁的同时使内存中inode的引用计数减1。

上面的实现方法是将路径名写在b0索引的那个block中，似乎也可以直接无视inode的数据结构，直接写在inode上，每个inode有64字节的空间，表示一个路径名足够了。

修改sys_open系统调用来支持打开符号链接文件

```c
uint64
sys_open(void)
{
  // ...
  int count = 0;
  while (ip->type == T_SYMLINK) {
    count ++;
    if (count > 10) {
      iunlockput(ip);
      end_op();
      return -1;
    }
    // printf("dead loop\n");
    char linkpath[MAXPATH];
    struct inode *tmp_ip;
    struct buf *bp;
    if (ip->addrs[0] == 0) {
      iunlockput(ip);
      end_op();
      return -1;
    }
    bp = bread(ip->dev, ip->addrs[0]);
    memmove(linkpath, bp->data, MAXPATH);
    brelse(bp);

    if ((tmp_ip = namei(linkpath)) == 0) {
      iunlockput(ip);
      end_op();
      return -1;
    }
    if (omode & O_NOFOLLOW) {
      break;
    } else {
      iunlockput(ip);
      ip = tmp_ip;
      ilock(ip);
    }
  }
  // ...
}
```
