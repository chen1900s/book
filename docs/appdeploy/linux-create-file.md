---
title: Linux创建指定大小文件
tags: Linux
categories: Linux
keywords: Linux
description: Linux创建指定大小文件
abbrlink: 6757b0f7
date: 2018-06-25 20:10:42
updated: 2018-06-25 23:58:58
---

在使用Linux时候，经常使用用 touch 命令创建一个空文件。当我们排除故障或想在某些特定场景中进行测试时，我们可能需要特定大小的大文件，比如1024MB 或5GB大小的文件，下面介绍几种常用的方法

### 使用 dd 命令创建大文件

dd 命令用于复制和转换文件

dd 命令是实际写入硬盘，文件产生的速度取决于硬盘的读写速度，根据文件的大小，该命令将需要一些时间才能完成。

假设我们要创建一个名为test.img 的 2 GB 大小的文本文件，可以执行以下操作：

```bash
dd if=/dev/zero of=test1.txt bs=2G count=1
```

```bash
[root@chen tmp]# dd if=/dev/zero of=text.img bs=2G count=1  #命令
0+1 records in
0+1 records out
2147479552 bytes (2.1 GB) copied, 94.4318 s, 22.7 MB/s
[root@cjweichen tmp]# ls -lh text.img        #查看文件大小
-rw-r--r-- 1 root root 2.0G Dec  8 10:46 text.img
```

我们可以根据需要来更改块大小和块数。例如，可以使用 bs=1M 和 count=1024 来获得 1024 Mb 的文件。

### 使用 truncate 命令创建大文件

truncate 命令将一个文件缩小或者扩展到所需大小。使用 -s 选项来指定文件的大小，执行速度会快些

我们使用 truncare 命令来创建一个 2GB 大小的文件。

```bash
truncate -s  2G test2.txt
```

可以使用`ls -lh test2.txt `命令查看生成的文件。

默认情况下，如果请求的输出文件不存在，truncate 命令将创建新文件。我们可以使用 -c 选项来避免创建新文件。

### 使用 fallocate 命令创建大文件

fallocate 命令创建大文件的方法，创建大文件的速度是三个中最快的。

假设我们要创建一个2 GB 的文件，可以执行以下操作：

```
fallocate -l 2G  test3.txt 
```

可以使用`ls -lh test3.txt `查看生成的文件。

### 结论

dd 和 truncate 创建的文件是稀疏文件。在计算机世界中，稀疏文件是一种特殊文件，具有不同的表观文件大小（它们可以扩展到的最大大小）和真实文件大小（为磁盘上的数据分配了多少空间）。

fallocate 命令则不会创建稀疏文件，而且它的速度更快，这也是我比较推荐使用 fallocate 创建大文件的原因。
