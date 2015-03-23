# Nginx中的进程间通信

## 1.Nginx中的channel通信机制

### 1.1概述

首先简单的说一下Nginx中channel通信的机制。

Nginx中的channel通信，本质上是多个进程之间,利用匿名套接字(socketpair)对来进行通信。

我们知道,socketpair可以创建出一对套接字,在这两个套接字的任何一个上面进行写操作,在另一个套接字上就可以相应的进行读操作,而且这个管道是全双工的。

那么,当父进程在调用了socketpair创建出一对匿名套接字对(A1,B1)后,fork出一个子进程,那么此时子进程也继承了这一对套接字对(A2,B2)。在这个基础上,父子进程即可进行通信了。例如,父进程对A1进行写操作,子进程可通过B2进行相应的读操作;子进程对B2进行写操作,父进程可以通过A1来进行相应的读操作等等。

我们假设,父进程依次fork了N个子进程,在每次fork之前,均如前所述调用了socketpair建立起一个匿名套接字对,这样,父进程与各个子进程之间即可通过各自的套接字对来进行通信。

但是各子进程之间能否使用匿名套接字对来进行通信呢？

我们假设父进程A中,它与子进程B之间的匿名套接字对为AB[2],它与子进程C之间的匿名套接字对为AC[2]。且进程B在进程C之前被fork出来。

对进程B而言，当它被fork出来后，它就继承了父进程创建的套接字对,命名为BA[2],这样父进程通过操作AB[2]，子进程B通过操作BA[2],即可实现父子进程之间的通信。

对进程C而言,当它被fork出来后，他就继承了父进程穿件的套接字对，命名为CA[2],这样父进程通过操作AC[2]，子进程C通过操作CA[2]，即可实现父子进程之间的通信。

但B和C有一点不同。由于B进程在C之前被fork，B进程无法从父进程中继承到父进程与C进程之间的匿名套接字对,而C进程在后面被fork出来，它却从父进程处继承到了父进程与子进程B之间的匿名套接字对。

这样，之后被fork出来的进程C,可以通过它从父进程那里继承到的与B进程相关联的匿名套接字对来向进程B发送消息，但进程B却无法向进程C发送消息。

当子进程数量比较多时，就会造成这样的情况：即后面的进程拥有前面每一个子进程的一个匿名套接字，但前面的进程则没有后面任何一个子进程的匿名套接字。

那么这个问题该如何解决呢？这就涉及到进程间传递文件描述符这个话题了。可以参考这里:[进程之间传递文件描述符](jin_cheng_jian_chuan_di_wen_jian_miao_shu_fu.md)。一个子进程被fork出来后，它可以依次向它之前被fork出来的所有子进程传递自己的描述符（匿名套接字对中的一个)。

通过这种机制,子进程之间也可以进行通信了。

Nginx中也就是这么做的。

### 1.2Nginx中的具体实现
在ngx_process.c中，定义了一个全局的数组ngx_processes:
```
ngx_process_t    ngx_processes[NGX_MAX_PROCESSES];
```
其中,ngx_process_t类型定义为:
```
typedef struct {
    ngx_pid_t           pid;
    int                 status;
    ngx_socket_t        channel[2];

    ngx_spawn_proc_pt   proc;
    void               *data;
    char               *name;

    unsigned            respawn:1;
    unsigned            just_spawn:1;
    unsigned            detached:1;
    unsigned            exiting:1;
    unsigned            exited:1;
} ngx_process_t;
```
在这里,我们只关心成员channel,

## 2.Nginx中的共享内存