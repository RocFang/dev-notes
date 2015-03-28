# Nginx如何控制某个特性是否打开

提到Nginx，大家首先会想到它的高性能，异步框架、模块化、upstream、红黑树等耳熟能详的技术实现。这些确实也是Nginx的核心，但作为一个优秀的开源项目，Nginx可以供我们借鉴的远不止这些，例如本文的话题：如何控制某个特性是否打开？

我们知道，在Linux下用源码安装方式编译安装一个软件时，标准情况下是有一个configure的动作，这个动作即是在编译前对环境进行检查，服务于后面的编译和安装，Nginx当然也不例外。

Nginx的configure文件是一个入口，在里面调用了很多其他脚本，这些脚本都位于源代码的auto目录下。本文重点涉及其中两个脚本：auto/have和auto/define.

它们的内容极其简单，分别如下：
1.auto/have:
```
# Copyright (C) Igor Sysoev
# Copyright (C) Nginx, Inc.


cat << END >> $NGX_AUTO_CONFIG_H

#ifndef $have
#define $have 1
#endif

END
```
2.auto/define
```
# Copyright (C) Igor Sysoev
# Copyright (C) Nginx, Inc.


cat << END >> $NGX_AUTO_CONFIG_H

#ifndef $have
#define $have $value
#endif

END
```
可见，两个脚本只有一处不同，是将have变量定义成value变量还是将其定义为1，所以后面仅仅以have脚本为例进行说明。

have脚本的用法：
```
have=XXXX  . auto/have
```
就会在$NGX\_AUTO\_CONFIG_H所代表的文件里（默认为objs/ngx\_auto\_config.h)追加如下内容：
```
#ifndef XXXX
#define XXXX 1
#endif
```

这样，就可以用XXXX宏是否定义来控制编译的过程。

下面以ngx\_memalign这个Nginx内部接口为例来进行详细的说明。
ngx\_memalign是Nginx里最基本的接口之一，经常会被调用。从接口的名字就可以看出，这个接口是用来处理内存分配和对齐的。其定义在ngx\_alloc.h中：
```
#if (NGX_HAVE_POSIX_MEMALIGN || NGX_HAVE_MEMALIGN)
void *ngx_memalign(size_t alignment, size_t size, ngx_log_t *log); 
#else 
#define ngx_memalign(alignment, size, log) ngx_alloc(size, log) 
#endif
```
上面代码的意图是，如果定义了NGX\_HAVE\_POSIX\_MEMALIGN宏或者NGX\_HAVE\_MEMALIGN宏，则声明函数ngx\_memalign，否则，简单的对ngx\_alloc进行一下封装（ngx\_alloc是对malloc的简单封装）。

我们在来看函数ngx\_memalign在ngx\_alloc.c中的定义：
```
#if (NGX_HAVE_POSIX_MEMALIGN)

void *
ngx_memalign(size_t alignment, size_t size, ngx_log_t *log)
{
    void *p;
    int err;

    err = posix_memalign(&p, alignment, size);

    if (err) {
        ngx_log_error(NGX_LOG_EMERG, log, err,
                      "posix_memalign(%uz, %uz) failed", alignment, size);
        p = NULL;
    }

    ngx_log_debug3(NGX_LOG_DEBUG_ALLOC, log, 0,
                   "posix_memalign: %p:%uz @%uz", p, size, alignment);

    return p;
}

#elif (NGX_HAVE_MEMALIGN)

void *
ngx_memalign(size_t alignment, size_t size, ngx_log_t *log)
{
    void *p;

    p = memalign(alignment, size);
    if (p == NULL) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "memalign(%uz, %uz) failed", alignment, size);
    }

    ngx_log_debug3(NGX_LOG_DEBUG_ALLOC, log, 0,
                   "memalign: %p:%uz @%uz", p, size, alignment);

    return p;
}

#endif
```
上面代码的意图为，如果定义了NGX\_HAVE\_POSIX\_MEMALIGN，那么ngx_memalign就是对posix\_memalign的简单封装，否则，如果定义了NGX\_HAVE\_MEMALIGN，则ngx\_memalign是对memalign的简单封装。

那么NGX\_HAVE\_POSIX\_MEMALIGN和NGX\_HAVE\_MEMALIGN又是什么时候被定义呢？

过程如下：
在auto/unix脚本，列出需要检查的接口，这里就是memalign和posix\_memalign了，交给auto/feature脚本来逐一处理，auto/feature脚本会对针对每一个带检查的接口，生成一个最基本的临时c源文件，该源文件会调用该接口。feature脚本然后对该源文件进行编译并判断最终生成的目标文件的可执行性。用这种动态检测的方法来判断该接口在当前系统中是否支持。

如果feature脚本验证出posix\_memalign和memalign接口在当前系统中都可用后，则会逐一调用have脚本：
```
have=$ngx_have_feature . auto/have
```
这里，变量ngx\_have\_feature即是NGX\_HAVE\_POSIX\_MEMALIGN和NGX\_HAVE\_MEMALIGN。

最终，have在configure、auto/unix、auto/feature、auto/have各个脚本的通力合作下，在objs/ngx\_auto\_config.h中就有了对NGX\_HAVE\_POSIX\_MEMALIGN和NGX\_HAVE\_MEMALIGN的定义，进而影响到ngx\_memalign接口的实现。

让我们从需求出发，将接口ngx\_memalign的需求描述一遍：

1. 如果系统支持posix\_memalign，则ngx\_memalign是对posix\_memalign的简单封装。
2. 如果系统不支持posix\_memalign，但支持memalign，则ngx\_memalign是对memalign的简单封装。
3. 如果系统竟然对posix\_memalign和memalign都不支持，则ngx\_memalign是对malloc的简单封装。

实现需求很简单，但如何实现的优雅，则是另一回事。Nginx向我们展示了，一个在C上尽力把服务器性能做到极致的项目，是如何在脚本上也尽最大努力做得漂亮。


