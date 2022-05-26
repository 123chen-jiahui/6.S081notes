## xv6上的文件系统（未完待续）

由于这部分内容比较繁杂，所以在这里整理一下

### 文件操作分层（从上到下）

#### 1. 进程所感知的文件层

能感知到目录、<font color = red>**逻辑**</font>文件

#### 2. 内核文件系统代码所操控的块层

能感知到磁盘布局、索引节点（inode）、目录项、盘块(<font color = red>**block**</font>)、块缓存

内核文件系统是以block为单位来操作的

#### 3. 设备所管理的设备层

能感知到IDE设备、扇区等

inode（指缓存到内存中的inode结构体，<font color = red>**注意dinode和inode的区别**</font>）中

+ nlink用于记录有多少个目录项指向它

+ ref用于记录多少个file结构体指向它（被打开了多少次）

索引节点的相关操作：

+ 对索引节点自身的**元数据**操作
  
  1. `struct inode *ialloc(uint dev, short type)`：<font color = red>**在磁盘上**</font>找到一个空闲的索引节点，将其设为allocated（通过log_write)，然后将其缓存到内存中（通过iget）
  
  2. `static struct inode *iget(uint dev, uint inum)`：跟bget差不多，将指定的索引节点从硬盘缓存带内存
  
  3. （<font color = red>**未完待续，视屏59分处**</font>

目录操作：在文件读写基础之上，将目录文件内容作为目录项处理：

+ 目录查找：
  
  1. `struct inode *dirlookup(struct inode *dp, char *name, uint *poff)`：在指定的inode dp内查找名字为name的目录项entry，然后对该目录项建立inode缓存，最后返回该缓存。显然，在这个函数中肯定要调用readi()来读取dp中的目录项。
  
  2. `static char *skipelem(cahr *path, char *name)`：也不知道怎么描述这个函数的功能，copy the next element from path into name, return a pointer to the element following the copied one.嗯，抄代码注释，很完美。

+ 文件定位（全路径）：
  
  1. `static struct inode *namex(char *path, int nameiparent, char *name)`：
  
  2. `namei()`
  
  3. `nameiparent()`（借助于namex()）
  
  4. `namex()`（借助于dirlookup()和skipelem()）

+ 创建和删除：`dirlink()`、删除

注意一个动词：<font color = red>**动态打开**</font>

文件操作的**系统调用**接口

+ 文件打开与关闭：`sys_open()`/`sys_close()`（只关闭动态打开的资源，不会管磁盘上的资源）/`sys_dup()`

+ 文件读写操作：`sys_read()`/`sys_write()`

+ 目录操作：`sys_mkdir()`/`sys_chdir()`/`sys_mknod()`/`sys_link()`/`sys_unlink()`

+ 其他操作：`sys_exec()`、`sys_pipe()`

<font color = red>**通过进程系统调用open打开一个文件的步骤：**</font>

1. 找到
