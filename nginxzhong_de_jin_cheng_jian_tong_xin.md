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
在这里,我们只关心成员channel成员，这个两元素的数组即用来存放一个匿名套接字对。

我们假设有程序运行后，有1个master进程和4个worker进程。那么，对这5个进程而言，每个进程都有一个4元素的数组ngx_process[4]，数组中每个元素都是一个ngx_process_t类型的结构体，包含了一个worker进程的相关信息。我们这里关心的是每个结构体的channel数组成员。

绘制成表如下:

| ngx_processes数组 | master | worker1 |worker2 | worker3 | worker4 |
| -- | -- | -- | -- | -- | -- |
| ngx_processes[0].channel | [x,x] | [x,x] | [x,x] | [x,x] | [x,x] |
| ngx_processes[1].channel | [x,x] | [x,x] | [x,x] | [x,x] | [x,x] |
| ngx_processes[2].channel | [x,x] | [x,x] | [x,x] | [x,x] | [x,x] |
| ngx_processes[3].channel | [x,x] | [x,x] | [x,x] | [x,x] | [x,x] |

上表的每一列表示每个进程的ngx_processes数组的各个元素的channel成员。

其中，master进程列中的每一个元素，表示master进程与对应的每个worker进程之间的匿名套接字对。

而每一个worker进程列中的每一个元素，表示该worker进程与对应的每个worker进程之间的匿名套接字对。当然这只是一个粗略的说法，与真实情况并不完全相符，还有很多细节需要进一步阐述。

我们直接借助《深入剖析Nginx》，直接看下图的实例:

| ngx_processes数组 | master | worker1 |worker2 | worker3 | worker4 |
| -- | -- | -- | -- | -- | -- |
| ngx_processes[0].channel | [3,7] | [**-1,7**] | [3,-1] | [3,-1] | [3,-1] |
| ngx_processes[1].channel | [8,9] | [3,0] | [**-1,9**] | [8,-1] | [8,-1] |
| ngx_processes[2].channel | [10,11] | [9,0] | [7,0] | [**-1,11**] | [10,-1] |
| ngx_processes[3].channel | [12,13] | [10,0] | [8,0] | [7,0] | [**-1,13**] |

再次感谢《深入剖析Nginx》的作者高群凯，觉得在这里我没法表达的比他更好了。所以下面会引用很多该书中的内容。

在上表中，每一个单元格的内容[a,b]分别表示channel[0]和channel[1]的值，-1表示这之前是描述符，但在之后被主动close()掉了，0表示这一直都无对应的描述符，其他数字表示对应的描述符值。

每一列数据都表示该列所对应进程与其他进程进行通信的描述符，如果当前列所对应进程为父进程，那么它与其它进程进行通信的描述符都为channel[0](其实channel[1]也可以)；如果当前列所对应的进程为子进程，那么它与父进程进行通信的描述符为channel[1]（注：这里书中说的太简略，应该为如果当前列所对应的进程为子进程，那么它与父进程进行通信的描述符为该进程的ngx_processes数组中，与本进程对应的元素中的channel[1]，在图中即为标粗的对角线部分，即[-1,7],[-1,9],[-1,11],[-1,13]这四对）。
## 2.Nginx中的共享内存