# Lab9: file system

## 实验目的

在本作业中，将增加xv6文件的最大大小。目前，xv6文件限制为268个块或`268*BSIZE`字节（在xv6中`BSIZE`为1024）。此限制来自以下事实：一个xv6 `inode`包含12个“直接”块号和一个“间接”块号，“一级间接”块指一个最多可容纳256个块号的块，总共12+256=268个块。将更改xv6文件系统代码，以支持每个`inode`中可包含256个一级间接块地址的“二级间接”块，每个一级间接块最多可以包含256个数据块地址。结果将是一个文件将能够包含多达65803个块，或256*256+256+11个块（11而不是12，因为我们将为二级间接块牺牲一个直接块号）。

## 实验过程

### Large files

1. 修改 `kernel/fs.h` 中的直接块号的宏定义 `NDIRECT` 为 11.
   根据实验要求, inode 中原本 12 个直接块号被修改为 了 11 个.

   ```c
   #define NDIRECT 11  // lab9-1
   ```

2. 修改 `inode` 相关结构体的块号数组. 具体包括 `kernel/fs.h` 中的磁盘 `inode` 结构体 `struct dinode`的 `addrs` 字段; 和 `kernel/file.h` 中的内存 `inode` 结构体 `struct inode` 的 `addrs` 字段. 将二者数组大小设置为 `NDIRECT+2`, 因为实际 inode 的块号总数没有改变, 但 `NDIRECT` 减少了 1.

   ```c
   // On-disk inode structure
   struct dinode {
     // ...
     uint addrs[NDIRECT+2];   // Data block addresses  // lab9-1
   };
   
   // in-memory copy of an inode
   struct inode {
     // ...
     uint addrs[NDIRECT+2];    // lab9-1
   };
   
   ```

3. 在 `kernel/fs.h` 中添加宏定义 `NDOUBLYINDIRECT`, 表示二级间接块号的总数, 类比 `NINDIRECT`. 由于是二级, 因此能够表示的块号应该为一级间接块号 `NINDIRECT` 的平方.

   ```c
   #define NDOUBLYINDIRECT (NINDIRECT * NINDIRECT)     // lab9-1
   ```

4. 修改 `kernel/fs.c` 中的 `bmap()` 函数.
   该函数用于返回 inode 的相对块号对应的磁盘中的块号.
   由于 inode 结构中前 `NDIRECT` 个块号与修改前是一致的, 因此只需要添加对第 `NDIRECT` 即 13 个块的二级间接索引的处理代码. 处理的方法与处理第 `NDIRECT` 个块号即一级间接块号的方法是类似的, 只是需要索引两次.

   ```c
   static uint
   bmap(struct inode *ip, uint bn)
   {
     uint addr, *a;
     struct buf *bp;
   
     if(bn < NDIRECT){
       if((addr = ip->addrs[bn]) == 0)
         ip->addrs[bn] = addr = balloc(ip->dev);
       return addr;
     }
     bn -= NDIRECT;
   
     if(bn < NINDIRECT){
       // Load indirect block, allocating if necessary.
       if((addr = ip->addrs[NDIRECT]) == 0)
         ip->addrs[NDIRECT] = addr = balloc(ip->dev);
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       if((addr = a[bn]) == 0){
         a[bn] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       return addr;
     }
   
     // doubly-indirect block - lab9-1
     bn -= NINDIRECT;
     if(bn < NDOUBLYINDIRECT) {
       // get the address of doubly-indirect block
       if((addr = ip->addrs[NDIRECT + 1]) == 0) {
         ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
       }
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       // get the address of singly-indirect block
       if((addr = a[bn / NINDIRECT]) == 0) {
         a[bn / NINDIRECT] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       bp = bread(ip->dev, addr);
       a = (uint*)bp->data;
       bn %= NINDIRECT;
       // get the address of direct block
       if((addr = a[bn]) == 0) {
         a[bn] = addr = balloc(ip->dev);
         log_write(bp);
       }
       brelse(bp);
       return addr;
     }
   
     panic("bmap: out of range");
   }
   ```

5. 修改 `kernel/fs.c` 中的 `itrunc()` 函数.
   该函数用于释放 inode 的数据块.
   由于添加了二级间接块的结构, 因此也需要添加对该部分的块的释放的代码. 释放的方式同一级间接块号的结构, 只需要两重循环去分别遍历二级间接块以及其中的一级间接块.

   ```c
   void
   itrunc(struct inode *ip)
   {
     int i, j, k;  // lab9-1
     struct buf *bp, *bp2;     // lab9-1
     uint *a, *a2; // lab9-1
   
     for(i = 0; i < NDIRECT; i++){
       if(ip->addrs[i]){
         bfree(ip->dev, ip->addrs[i]);
         ip->addrs[i] = 0;
       }
     }
   
     if(ip->addrs[NDIRECT]){
       bp = bread(ip->dev, ip->addrs[NDIRECT]);
       a = (uint*)bp->data;
       for(j = 0; j < NINDIRECT; j++){
         if(a[j])
           bfree(ip->dev, a[j]);
       }
       brelse(bp);
       bfree(ip->dev, ip->addrs[NDIRECT]);
       ip->addrs[NDIRECT] = 0;
     }
     // free the doubly-indirect block - lab9-1
     if(ip->addrs[NDIRECT + 1]) {
       bp = bread(ip->dev, ip->addrs[NDIRECT + 1]);
       a = (uint*)bp->data;
       for(j = 0; j < NINDIRECT; ++j) {
         if(a[j]) {
           bp2 = bread(ip->dev, a[j]);
           a2 = (uint*)bp2->data;
           for(k = 0; k < NINDIRECT; ++k) {
             if(a2[k]) {
               bfree(ip->dev, a2[k]);
             }
           }
           brelse(bp2);
           bfree(ip->dev, a[j]);
           a[j] = 0;
         }
       }
       brelse(bp);
       bfree(ip->dev, ip->addrs[NDIRECT + 1]);
       ip->addrs[NDIRECT + 1] = 0;
     }
   
     ip->size = 0;
     iupdate(ip);
   }
   ```

6. 修改 `kernel/fs.h` 中的文件最大大小的宏定义 `MAXFILE`. 由于添加了二级间接块的结构, xv6 支持的文件大小的上限自然增大, 此处要修改为正确的值.

   ```c
   #define MAXFILE (NDIRECT + NINDIRECT + NDOUBLYINDIRECT) // lab9-1
   ```

   ![image-20240719175533550](.\images\image-20240719175533550.png)

   ![image-20240719175548168](.\images\image-20240719175548168.png)

   ### Symbolic link

   **要求**：在 xv6 操作系统中实现符号链接（软链接）的功能

   1. 添加有关 `symlink` 系统调用的定义声明. 包括 `kernel/syscall.h`, `kernel/syscall.c`, `user/usys.pl` 和 `user/user.h`.

   2. 添加新的文件类型 `T_SYMLINK` 到 `kernel/stat.h` 中.

   3. 添加新的文件标志位 `O_NOFOLLOW` 到 `kernel/fcntl.h` 中.

   4. 在 `kernel/sysfile.c` 中实现 `sys_symlink()` 函数.
      该函数即用来生成符号链接. 符号链接相当于一个特殊的独立的文件, 其中存储的数据即目标文件的路径.
      因此在该函数中, 首先通过 `create()` 创建符号链接路径对应的 inode 结构(同时使用 `T_SYMLINK` 与普通的文件进行区分). 然后再通过 `writei()` 将链接的目标文件的路径写入 inode (的 block)中即可. 在这个过程中, 无需判断连接的目标路径是否有效.

   5. 修改 `kernel/sysfile` 的 `sys_open()` 函数.
      该函数使用来打开文件的, 对于符号链接一般情况下需要打开的是其链接的目标文件, 因此需要对符号链接文件进行额外处理.

      ![image-20240719183910442](.\images\image-20240719183910442.png)

## 实验中遇到的问题及解决方案

**问题**： xv6 中执行 `symlinktest` 会出现 `Failed to open 1` 的错误

**解决方法**：对成环的代码进行检测发现了问题所在, **开始记录的不是 inode number 而是执行 inode 的指针, 并通过指针进行比较**. 而需要注意的一点是, **这里的 `struct inode` 实际上是内存中的 inode 缓存**, 当调用 `iput()` 在 `ref` **引用计数为 0 时实际上该 inode 被回收**后续可以通过 `iget()` 存储磁盘中的 inode 信息(相当于有个内存 inode 的缓存池).

## 实验心得

文件系统实验中，我深入理解了文件系统的内部工作原理，特别是如何在磁盘上组织和管理数据。通过实现和调试各种文件操作，如创建、读取和写入文件，我掌握了inode结构和块管理的概念。这个实验不仅增强了我对操作系统中文件系统模块的理解，还培养了我的调试能力和解决问题的思维方式，进一步体会到操作系统设计的复杂性和精妙之处。

