## 1. 什么叫惊群现象

首先，我们看看[维基百科对惊群的定义][1]:

>The thundering herd problem occurs when a large number of processes waiting for an event are awoken when that event occurs, but only one process is able to proceed at a time. After the processes wake up, they all demand the resource and a decision must be made as to which process can continue. After the decision is made, the remaining processes are put back to sleep, only to all wake up again to request access to the resource.
>
>This occurs repeatedly, until there are no more processes to be woken up. Because all the processes use system resources upon waking, it is more efficient if only one process was woken up at a time.
>
>This may render the computer unusable, but it can also be used as a technique if there is no other way to decide which process should continue (for example when programming with semaphores).
	
简而言之，惊群现象（thundering herd）就是当多个进程和线程在同时阻塞等待同一个事件时，如果这个事件发生，会唤醒所有的进程，但最终只可能有一个进程/线程对该事件进行处理，其他进程/线程会在失败后重新休眠，这种性能浪费就是惊群。


<!--more-->


	
## 2. accept 惊群

考虑如下场景：
主进程创建socket, bind, listen之后，fork出多个子进程，每个子进程都开始循环处理（accept)这个socket。每个进程都阻塞在accpet上，当一个新的连接到来时，所有的进程都会被唤醒，但其中只有一个进程会accept成功，其余皆失败，重新休眠。这就是accept惊群。
	
那么这个问题真的存在吗？
	
事实上，历史上，Linux 的 accpet 确实存在惊群问题，但现在的内核都解决该问题了。即，当多个进程/线程都阻塞在对同一个 socket 的 accept 调用上时，当有一个新的连接到来，内核只会唤醒一个进程，其他进程保持休眠，压根就不会被唤醒。
	
测试代码如下：

```
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <sys/wait.h>
#include <stdio.h>
#include <string.h>
#define PROCESS_NUM 10
int main()
{
    int fd = socket(PF_INET, SOCK_STREAM, 0);
    int connfd;
    int pid;
    char sendbuff[1024];
	struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serveraddr.sin_port = htons(1234);
	bind(fd, (struct sockaddr*)&serveraddr, sizeof(serveraddr));
    listen(fd, 1024);
	        int i;
    for(i = 0; i < PROCESS_NUM; i++)
    {
        int pid = fork();
        if(pid == 0)
        {
            while(1)
            {
                connfd = accept(fd, (struct sockaddr*)NULL, NULL);
                snprintf(sendbuff, sizeof(sendbuff), "accept PID is %d\n", getpid());
                
                send(connfd, sendbuff, strlen(sendbuff) + 1, 0);
                printf("process %d accept success!\n", getpid());
                close(connfd);
            }
        }
    }
	        int status;
    wait(&status);
    return 0;
}
```	
	
当我们对该服务器发起连接请求（用 telnet/curl 等模拟）时，会看到只有一个进程被唤醒。
	
关于 accept 惊群的一些帖子或文章：

* [惊群问题在 linux 上可能是莫须有的问题][2]
* [Does the Thundering Herd Problem exist on Linux anymore?][3]
* [历史上解决 linux accept 惊群的补丁讨论][4]

## 3. epoll惊群

如上所述，accept 已经不存在惊群问题，但 epoll 上还是存在惊群问题。即，如果多个进程/线程阻塞在监听同一个 listening socket fd 的 epoll_wait 上，当有一个新的连接到来时，所有的进程都会被唤醒。
	
考虑如下场景：
	
主进程创建 socket, bind， listen 后，将该 socket 加入到 epoll 中，然后 fork 出多个子进程，每个进程都阻塞在 epoll_wait 上，如果有事件到来，则判断该事件是否是该 socket 上的事件，如果是，说明有新的连接到来了，则进行 accept 操作。为了简化处理，忽略后续的读写以及对 accept 返回的新的套接字的处理，直接断开连接。
	
那么，当新的连接到来时，是否每个阻塞在 epoll_wait 上的进程都会被唤醒呢？
	
很多博客中提到，测试表明虽然 epoll_wait 不会像 accept 那样只唤醒一个进程/线程，但也不会把所有的进程/线程都唤醒。例如这篇文章：[关于多进程 epoll 与 “惊群”问题][5]。

为了验证这个问题，我自己写了一个测试程序：

```
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netdb.h>
#include <string.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/wait.h>
#define PROCESS_NUM 10
static int
create_and_bind (char *port)
{
    int fd = socket(PF_INET, SOCK_STREAM, 0);
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
    serveraddr.sin_port = htons(atoi(port));
    bind(fd, (struct sockaddr*)&serveraddr, sizeof(serveraddr));
    return fd;
}
	static int
make_socket_non_blocking (int sfd)
{
    int flags, s;
 
    flags = fcntl (sfd, F_GETFL, 0);
    if (flags == -1)
    {
        perror ("fcntl");
        return -1;
    }
 
    flags |= O_NONBLOCK;
    s = fcntl (sfd, F_SETFL, flags);
    if (s == -1)
    {
        perror ("fcntl");
        return -1;
    }
 
    return 0;
}
	#define MAXEVENTS 64
 
int
main (int argc, char *argv[])
{
    int sfd, s;
    int efd;
    struct epoll_event event;
    struct epoll_event *events;
 
    sfd = create_and_bind("1234");
    if (sfd == -1)
        abort ();
 
    s = make_socket_non_blocking (sfd);
    if (s == -1)
        abort ();
 
    s = listen(sfd, SOMAXCONN);
    if (s == -1)
    {
        perror ("listen");
        abort ();
    }
 
    efd = epoll_create(MAXEVENTS);
    if (efd == -1)
    {
        perror("epoll_create");
        abort();
    }
 
    event.data.fd = sfd;
    //event.events = EPOLLIN | EPOLLET;
    event.events = EPOLLIN;
    s = epoll_ctl(efd, EPOLL_CTL_ADD, sfd, &event);
    if (s == -1)
    {
        perror("epoll_ctl");
        abort();
    }
 
    /* Buffer where events are returned */
    events = calloc(MAXEVENTS, sizeof event);
	        int k;
    for(k = 0; k < PROCESS_NUM; k++)
    {
        int pid = fork();
        if(pid == 0)
        {
 
            /* The event loop */
            while (1)
            {
                int n, i;
                n = epoll_wait(efd, events, MAXEVENTS, -1);
                printf("process %d return from epoll_wait!\n", getpid());
	                                   /* sleep here is very important!*/
                //sleep(2);
	                                   for (i = 0; i < n; i++)
                {
                    if ((events[i].events & EPOLLERR) || (events[i].events & EPOLLHUP) || (!(events[i].events &                                    EPOLLIN)))
                    {
                        /* An error has occured on this fd, or the socket is not
                        ready for reading (why were we notified then?) */
                        fprintf (stderr, "epoll error\n");
                        close (events[i].data.fd);
                        continue;
                    }
                    else if (sfd == events[i].data.fd)
                    {
                        /* We have a notification on the listening socket, which
                        means one or more incoming connections. */
                        struct sockaddr in_addr;
                        socklen_t in_len;
                        int infd;
                        char hbuf[NI_MAXHOST], sbuf[NI_MAXSERV];
 
                        in_len = sizeof in_addr;
                        infd = accept(sfd, &in_addr, &in_len);
                        if (infd == -1)
                        {
                            printf("process %d accept failed!\n", getpid());
                            break;
                        }
                        printf("process %d accept successed!\n", getpid());
 
                        /* Make the incoming socket non-blocking and add it to the
                        list of fds to monitor. */
                        close(infd); 
                    }
                }
            }
        }
    }
    int status;
    wait(&status);
    free (events);
    close (sfd);
    return EXIT_SUCCESS;
}
```

发现确实如上面那篇博客里所说，当我模拟发起一个请求时，只有一个或少数几个进程被唤醒了。
	
也就是说，到目前为止，还没有得到一个确定的答案。但后来，在下面这篇博客中看到这样一个评论：http://blog.csdn.net/spch2008/article/details/18301357

>这个总结，需要进一步阐述，你的实验，看上去是只有4个进程唤醒了，而事实上，其余进程没有被唤醒的原因是你的某个进程已经处理完这个 accept，内核队列上已经没有这个事件，无需唤醒其他进程。你可以在 epoll 获知这个 accept 事件的时候，不要立即去处理，而是 sleep 下，这样所有的进程都会被唤起
	
看到这个评论后，我顿时如醍醐灌顶，重新修改了上面的测试程序，即在 epoll_wait 返回后，加了个 sleep 语句，这时再测试，果然发现所有的进程都被唤醒了。
	
所以，epoll_wait上的惊群确实是存在的。
	
## 4. 为什么内核不处理 epoll 惊群

看到这里，我们可能有疑惑了，为什么内核对 accept 的惊群做了处理，而现在仍然存在 epoll 的惊群现象呢？

我想，应该是这样的：
accept 确实应该只能被一个进程调用成功，内核很清楚这一点。但 epoll 不一样，他监听的文件描述符，除了可能后续被 accept 调用外，还有可能是其他网络 IO 事件的，而其他 IO 事件是否只能由一个进程处理，是不一定的，内核不能保证这一点，这是一个由用户决定的事情，例如可能一个文件会由多个进程来读写。所以，对 epoll 的惊群，内核则不予处理。
	
## 5. Nginx 是如何处理惊群问题的

在思考这个问题之前，我们应该以前对前面所讲几点有所了解，即先弄清楚问题的背景，并能自己复现出来，而不仅仅只是看书或博客，然后再来看看 Nginx 的解决之道。这个顺序不应该颠倒。

首先，我们先大概梳理一下 Nginx 的网络架构，几个关键步骤为:

1. Nginx 主进程解析配置文件，根据 listen 指令，将监听套接字初始化到全局变量 ngx_cycle 的 listening 数组之中。此时，监听套接字的创建、绑定工作早已完成。
2. Nginx 主进程 fork 出多个子进程。
3. 每个子进程在 ngx_worker_process_init 方法里依次调用各个 Nginx 模块的 init_process 钩子，其中当然也包括 NGX_EVENT_MODULE 类型的 ngx_event_core_module 模块，其 init_process 钩子为 ngx_event_process_init。
4. ngx_event_process_init 函数会初始化 Nginx 内部的连接池，并把 ngx_cycle 里的监听套接字数组通过连接池来获得相应的表示连接的 ngx_connection_t 数据结构，这里关于 Nginx 的连接池先略过。我们主要看 ngx_event_process_init 函数所做的另一个工作：如果在配置文件里**没有**开启[accept_mutex锁][6]，就通过 ngx_add_event 将所有的监听套接字添加到 epoll 中。
5. 每一个 Nginx 子进程在执行完 ngx_worker_process_init 后，会在一个死循环中执行 ngx_process_events_and_timers，这就进入到时间处理的核心逻辑了。
6. 在 ngx_process_events_and_timers 中，如果在配置文件里开启了 accept_mutext 锁，子进程就会去获取 accet_mutext 锁。如果获取成功，则通过 ngx_enable_accept_events 将监听套接字添加到 epoll 中，否则，不会将监听套接字添加到 epoll 中，甚至有可能会调用 ngx_disable_accept_events 将监听套接字从 epoll 中删除（如果在之前的连接中，本worker子进程已经获得过accept_mutex锁)。
7. ngx_process_events_and_timers 继续调用 ngx_process_events，在这个函数里面阻塞调用 epoll_wait。

至此，关于 Nginx 如何处理 fork 后的监听套接字，我们已经差不多理清楚了，当然还有一些细节略过了，比如在每个 Nginx 在获取 accept_mutex 锁前，还会根据当前负载来判断是否参与 accept_mutex 锁的争夺。

把这个过程理清了之后，Nginx 解决惊群问题的方法也就出来了，就是利用 accept_mutex 这把锁。

如果配置文件中没有开启 accept_mutex，则所有的监听套接字不管三七二十一，都加入到 epoll中，这样当一个新的连接来到时，所有的 worker 子进程都会惊醒。

如果配置文件中开启了 accept_mutex，则只有一个子进程会将监听套接字添加到 epoll 中，这样当一个新的连接来到时，当然就只有一个 worker 子进程会被唤醒了。

## 6. 小结

现在我们对惊群及 Nginx 的处理总结如下：

* accept 不会有惊群，epoll_wait 才会。
* Nginx 的 accept_mutex,并不是解决 accept 惊群问题，而是解决 epoll_wait 惊群问题。
* 说Nginx 解决了 epoll_wait 惊群问题，也是不对的，它只是控制是否将监听套接字加入到epoll 中。监听套接字只在一个子进程的 epoll 中，当新的连接来到时，其他子进程当然不会惊醒了。
	
## 7. 其他参考文章
[“惊群”，看看 nginx 是怎么解决它的][7]

  [1]: http://en.wikipedia.org/wiki/Thundering_herd_problem
  [2]: http://bbs.chinaunix.net/thread-946261-1-1.html
  [3]: http://stackoverflow.com/questions/2213779/does-the-thundering-herd-problem-exist-on-linux-anymore
  [4]: http://www.citi.umich.edu/projects/linux-scalability/reports/accept.html
  [5]: http://blog.163.com/pandalove@126/blog/static/9800324520122633515612
  [6]: http://nginx.org/en/docs/ngx_core_module.html#accept_mutex
  [7]: http://blog.csdn.net/russell_tao/article/details/7204260