---
layout: post
title: "Linux Page Cache的理解"
category: Linux
---
因为硬盘和内存的读写性能差距巨大，Linux默认情况是以异步方式读写文件的。比如调用系统函数open()打开或者创建文件时缺省情况下是带有O_ASYNC flag的。Linux借助于内核的page cache来实现这种异步操作。引用《Understanding the Linux Kernel, 3rd Edition》中关于`page cache`的定义：
>The page cache is the main disk cache used by the Linux kernel. In most cases, the kernel refers to the page cache when reading from or writing to disk. New pages are added to the page cache to satisfy User Mode processes's read requests. If the page is not already in the cache, a new entry is added to the cache and filled with the data read from the disk. If there is enough free memory, the page is kept in the cache for an indefinite period of time and can then be reused by other processes without accessing the disk.  
Similarly, before writing a page of data to a block device, the kernel verifies whether the corresponding page is already included in the cache; if not, a new entry is added to the cache and filled with the data to be written on disk. The I/O data transfer does not start immediately: the disk update is delayed for a few seconds, thus giving a chance to the processes to further modify the data to be written (in other words, the kernel implements deferred write operations).

也就是说，我们平常向硬盘写文件时，默认异步情况下，并不是直接把文件内容写入到硬盘中才返回的，而是成功拷贝到内核的page cache后就直接返回，所以大多数情况下，硬盘写操作不会是性能瓶颈。写入到内核page cache的pages成为dirty pages，稍后会由内核线程pdflush真正写入到硬盘上。

从硬盘读取文件时，同样不是直接把硬盘上文件内容读取到用户态内存，而是先拷贝到内核的page cache，然后再“拷贝”到用户态内存，这样用户就可以访问该文件。因为涉及到硬盘操作，所以第一次读取一个文件时，不会有性能提升；不过，如果一个文件已经存在page cache中，再次读取该文件时就可以直接从page cache中命中读取不涉及硬盘操作，这时性能就会有很大提高。

下面用`dd`比较下异步（缺省模式）和同步写硬盘的速度差别：
{% highlight c++ %}
$ dd if=/dev/urandom of=async.txt bs=64M count=16 iflag=fullblock
16+0 records in
16+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 7.618 s, 141 MB/s
$ dd if=/dev/urandom of=sync.txt bs=64M count=16 iflag=fullblock oflag=sync
16+0 records in
16+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 13.2175 s, 81.2 MB/s
{% endhighlight %}

page cache除了可以提升和硬盘交互性能外，下面继续讨论page cache功能。
### 1. 如果程序crash，异步模式会丢失数据吗？
比如存在这样的场景：一批数据已经成功写入到page cache，这时程序突然crash，但是在page cache里的数据还没来得及被pdflush写回到硬盘，这批数据会丢失吗？  
答案是，要看具体情况：
1. 如果OS没有crash或者重启的话，仅仅是写数据的程序crash，那么已经成功写入到page cache中的dirty pages是会被pdflush在合适的时机被写回到硬盘，不会丢失数据；
2. 如果OS也crash或者重启的话，因为page cache存放在内存中，一旦断电就丢失了，那么就会丢失数据。  
至于这种情况下，会丢失多少数据，主要看系统重启前有多少dirty pages被写入到硬盘，已经成功写回硬盘的就不会丢失；没来得急写回硬盘的数据就彻底丢失了。这也是异步写硬盘的一个潜在风险。  
同步写硬盘时就不存在这种丢数据的风险。同步写操作返回成功时，能保证数据一定被保存在硬盘上了。

引用RocksDB wiki中关于“[Asynchronous Writes]”描述：
>Asynchronous writes are often more than a thousand times as fast as synchronous writes. The downside of asynchronous writes is that a crash of the machine may cause the last few updates to be lost. Note that a crash of just the writing process (i.e., not a reboot) will not cause any loss since even when sync is false, an update is pushed from the process memory into the operating system before it is considered done.

那么如何避免因为系统重启或者机器突然断电，导致数据丢失问题呢？  
可以借助于WAL（Write-Ahead Log）技术。

WAL技术在数据库系统中比较常见，在数据库中一般又称之为redo log，Linux 文件系统ext3/ext4称之为journaling。WAL作用是：写数据库或者文件系统前，先把相关的metadata和文件内容写入到WAL日志中，然后才真正写数据库或者文件系统。WAL日志是append模式，所以，对WAL日志的操作要比对数据库或者文件系统的操作轻量级得多。如果对WAL日志采用同步写模式，那么WAL日志写成功，即使写数据库或者文件系统失败，可以用WAL日志来恢复数据库或者文件系统里的文件。

### 2. 查看一个文件占用page cache情况
可以借助于[vmtouch]工具（网上提到的linux-ftools中fincore已经停止维护，vmtouch功能类似）：
>vmtouch is a tool for learning about and controlling the file system cache of unix and unix-like systems.

比如，有一个文件sample.txt:
{% highlight c++ %}
$ ll sample.txt
-rw-r--r-- 1 root root 660000271 Mar 14 15:59 sample.txt
$ ll -h sample.txt
-rw-r--r-- 1 root root 630M Mar 14 15:59 sample.txt
{% endhighlight %}
如果这个sample.txt文件从来没有被打开过，结果如下：
{% highlight c++ %}
$ vmtouch sample.txt
           Files: 1
     Directories: 0
  Resident Pages: 0/161133  0/629M  0%
         Elapsed: 0.003136 seconds
{% endhighlight %}
接着用vim打开这个sample.txt文件，因为这个文件比较大，所以vim会hang住一段时间。同时可以用`watch -n 1 vmtouch sample.txt`持续观察wmtouch输出，可以看到“Resident Pages”的值持续增长直到100%，这时vim也结束了hang的过程并接受用户操作。最终结果是：
{% highlight c++ %}
$ vmtouch sample.txt
           Files: 1
     Directories: 0
  Resident Pages: 161133/161133  629M/629M  100%
         Elapsed: 0.022034 seconds
{% endhighlight %}

参看资料：
* Understanding the Linux Kernel, 3rd Edition
* [vmtouch]
* [Page Cache, the Affair Between Memory and Files]

[Asynchronous Writes]: https://github.com/facebook/rocksdb/wiki/Basic-Operations#asynchronous-writes
[vmtouch]: https://hoytech.com/vmtouch/
[Page Cache, the Affair Between Memory and Files]: https://manybutfinite.com/post/page-cache-the-affair-between-memory-and-files/