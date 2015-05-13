# vfork 挂掉的一个问题

**注意：本文并非原创，转自陈皓的博客:[vfork挂掉的一个问题](http://coolshell.cn/articles/12103.html)，版权归陈皓所有。**

在知乎上，有个人问了这样的一个问题——[为什么vfork的子进程里用return，整个程序会挂掉，而且exit()不会？](http://www.zhihu.com/question/26591968)并给出了如下的代码，下面的代码一运行就挂掉了，但如果把子进程的return改成exit(0)就没事。

我受邀后本来不想回答这个问题的，因为这个问题明显就是RTFM的事，后来，发现这个问题放在那里好长时间，而挂在下面的几个答案又跑偏得比较严重，我觉得可能有些朋友看到那样的答案会被误导，所以就上去回答了一下这个问题。

下面我把问题和我的回答发布在这里，也供更多的人查看。

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int main(void) {
    int var;
    var = 88;
    if ((pid = vfork()) < 0) {
        printf("vfork error");
        exit(-1);
    } else if (pid == 0) { /* 子进程 */
        var++;
        return 0;
    }
    printf("pid=%d, glob=%d, var=%d\n", getpid(), glob, var);
    return 0;
}

```

## 基础知识

首先说一下fork和vfork的差别：

* fork 是 创建一个子进程，并把父进程的内存数据copy到子进程中。
* vfork是 创建一个子进程，并和父进程的内存数据share一起用。
这两个的差别是，一个是copy，一个是share。（关于fork，可以参看酷壳之前的《[一道fork的面试题](http://coolshell.cn/articles/7965.html)》）

你 man vfork 一下，你可以看到，vfork是这样的工作的，

1. 保证子进程先执行。
2. 当子进程调用exit()或exec()后，父进程往下执行。

那么，为什么要干出一个vfork这个玩意？ 原因在man page也讲得很清楚了：

>Historic Description
>
>Under Linux, fork(2) is implemented using copy-on-write pages, so the only penalty incurred by fork(2) is the time and memory required to duplicate the parent’s page tables, and to create a unique task structure for the child. However, in the bad old days a fork(2) would require making a complete copy of the caller’s data space, often needlessly, since usually immediately afterwards an exec(3) is done. Thus, for greater efficiency, BSD introduced the vfork() system call, which did not fully copy the address space of the parent process, but borrowed the parent’s memory and thread of control until a call to execve(2) or an exit occurred. The parent process was suspended while the child was using its resources. The use of vfork() was tricky: for example, not modifying data in the parent process depended on knowing which variables are held in a register.

意思是这样的—— 起初只有fork，但是很多程序在fork一个子进程后就exec一个外部程序，于是fork需要copy父进程的数据这个动作就变得毫无意了，而且这样干还很重（注：后来，fork做了优化，详见本文后面），所以，BSD搞出了个父子进程共享的 vfork，这样成本比较低。因此，vfork本就是为了exec而生。

## 为什么return会挂掉，exit()不会？

从上面我们知道，结束子进程的调用是exit()而不是return，如果你在vfork中return了，那么，这就意味main()函数return了，注意因为函数栈父子进程共享，所以整个程序的栈就跪了。

如果你在子进程中return，那么基本是下面的过程：

1. 子进程的main() 函数 return了，于是程序的函数栈发生了变化。
2. 而main()函数return后，通常会调用 exit()或相似的函数（如：_exit()，exitgroup()）
3. 这时，父进程收到子进程exit()，开始从vfork返回，但是尼玛，老子的栈都被你子进程给return干废掉了，你让我怎么执行？（注：栈会返回一个诡异一个栈地址，对于某些内核版本的实现，直接报“栈错误”就给跪了，然而，对于某些内核版本的实现，于是有可能会再次调用main()，于是进入了一个无限循环的结果，直到vfork 调用返回 error）

好了，现在再回到 return 和 exit，return会释放局部变量，并弹栈，回到上级函数执行。exit直接退掉。如果你用c++ 你就知道，return会调用局部对象的析构函数，exit不会。（注：exit不是系统调用，是glibc对系统调用 _exit()或_exitgroup()的封装）

可见，子进程调用exit() 没有修改函数栈，所以，父进程得以顺利执行。

## 关于fork的优化

很明显，fork太重，而vfork又太危险，所以，就有人开始优化fork这个系统调用。优化的技术用到了著名的写时拷贝（COW）。

也就是说，对于fork后并不是马上拷贝内存，而是只有你在需要改变的时候，才会从父进程中拷贝到子进程中，这样fork后立马执行exec的成本就非常小了。所以，Linux的Man Page中并不鼓励使用vfork() ——

>“ It is rather unfortunate that Linux revived this specter from the past. The BSD man page states: “This system call will be eliminated when proper system sharing mechanisms are implemented. Users should not depend on the memory sharing semantics of vfork() as it will, in that case, be made synonymous to fork(2).””

于是，从BSD4.4开始，他们让vfork和fork变成一样的了

但在后来，NetBSD 1.3 又把传统的vfork给捡了回来，说是vfork的性能在 Pentium Pro 200MHz 的机器（这机器好古董啊）上有可以提高几秒钟的性能。详情见——“[NetBSD Documentation: Why implement traditional vfork()](http://www.netbsd.org/docs/kernel/vfork.html)”

今天的Linux下，fork和vfork还是各是各的，不过，还是建议你不要用vfork，除非你非常关注性能。

（全文完）

（转载本站文章请注明作者和出处 酷 壳 – CoolShell.cn ，请勿用于任何商业用途）