# Nginx中的进程间通信

## 1.Nginx中的channel通信机制

首先简单的说一下Nginx中channel通信的机制。

Nginx中的channel通信，本质上是多个进程之间,利用匿名套接字(socketpair)对来进行通信。

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
在这里,我们只关心成员channel。

## 2.Nginx中的共享内存