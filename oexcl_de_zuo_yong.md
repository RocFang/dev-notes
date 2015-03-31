# O_EXCL的作用

## 1.原始语义

与O\_CREATE标志组合起来调用open，确保指定的文件由open的调用者创建，否则返回错误。即，如果进程A用O\_CREATE和O\_EXCL标志来调用open，期望创建一个指定的文件file1,如果file1不存在，则open成功返回且创建file1，如果file1已经存在了（即不是由进程A创建的），那么open返回错误。

## 2.使用场景
O\_CREATE|O\_EXCL多用于确保一个一个程序只能执行单个进程，不能执行多个进程。原理如下，假设进程A是某程序的一个实例，如果它用O\_CREATE|O\_EXCL标志能够成功创建指定的文件，说明它是该程序的唯一实例，可以继续执行；如果返回错误，说明该文件已经存在，进而说明系统中已经运行着一个该程序的其它实例，检测到错误的返回值后，该实例就可以退出了。

**之所以能这么用的唯一理由是该操作是原子的**。

之所以这么说，理由如下。假设同样语义的非原子的操作流程如下:
```
if( access(file, R_OK) == -1 )   /* 首先检查文件是否存在 */
    open(file, O_RDWR | O_CREAT，0666);  /* 如果不存在，那我创建一个这样的文件 */
...  /* 继续执行任务 */
```

由于判断文件是否存在与创建文件是两个步骤，就会存在临界竞争的问题。试想下面的场景：

1. 某程序的进程A判断文件不存在，因此A认为自己是此时系统中该程序唯一的实例，准备继续执行创建指定的文件。
2. 操作系统的调度策略恰好在此时起作用，进程A暂停执行。此时指定的文件还没有创建。
3. 该程序的另一个进程B开始执行，同样，由于指定的文件不存在，B也认为自己是此时系统中该程序的唯一实例，准备继续执行创建指定的文件。
4. 这样，进程A和进程B都能成功调用open并继续往下执行。
5. 此时系统中就同时运行着该程序的两个实例，与仅运行一个进程的期望不符。

所以说，O\_EXCL与O\_CREATE联合使用的前提就是该操作是原子的。
## 3.非原子操作如何达到同样目的
假设现在不能使用O_EXCL|O_CREATE，或者假设用O_EXCL|O_CREATE调用open并不是原子的，该如何达到上面关于“一个程序只能运行一个实例”的要求呢？

可以用系统调用link来实现。

link的原型如下:
```
 #include <unistd.h>

 int link(const char *oldpath, const char *newpath);
```
link的作用是为oldpath指定的源文件创建一个newpath指定的链接文件(硬链接，hard link)。如果创建成功，则返回0，如果newpath路径指定的目标文件在调用link前已经存在，则link会错误返回。

根据link的特点，可以达到上面的要求：
1. 首先确保文件系统中已经有一个源文件file1。
2. 某程序的一个进程A开始执行，调用link，试图创建一个file1的链接文件file2。
3. 如果A调用link成功，说明该程序此时只有进程A锁定了该文件，进程A可以继续往下执行。
4. 由于进程A为file1创建了一个链接文件file2,此时file1的链接数是2（用stat可获取链接数）。
5. 进程B调用link,同样试图创建file1的链接文件file2，但由于file2已经存在，link错误返回。可进一步调用stat系统调用，查看file1的链接数，确定该链接数是2。
6. 进程B退出。

## 4. O\_EXCL|O\_CREATE确实有可能是非原子的
在NFS上，O\_EXCL|O\_CREATE确实有可能是非原子的：
>On NFS, O\_EXCL is supported only when using NFSv3 or later on
              kernel 2.6 or later.  In NFS environments where O\_EXCL support
              is not provided, programs that rely on it for performing
              locking tasks will contain a race condition.

在这种情况下，上面讨论的使用link的方法就有了用武之地。如上引文所述，在最新的内核中，NFS中并不存在该问题，用O\_EXCL|O\_CREATE仍然能满足要求。

## 5.其它
O\_EXCL一般只能和O_CREATE一起使用，不能单独使用，但有一个例外：在2.6即以后的内核中，如果open指定的文件是一个块设备文件，O\_EXCL可以单独使用，此时，如果该块设备正在被使用（例如已经被挂在)，那么open将失败返回，错误码是EBUSY;除此之外，单独使用O\_EXCL的后果无法预知：
>In general, the behavior of O\_EXCL is undefined if it is used
              without O\_CREAT.  There is one exception: on Linux 2.6 and
              later, O\_EXCL can be used without O\_CREAT if pathname refers
              to a block device.  If the block device is in use by the
              system (e.g., mounted), open() fails with the error EBUSY.