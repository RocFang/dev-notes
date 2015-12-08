# Nginx对Connection头的处理过程

### 1. 标准

RFC2616 中，对 Connection 的说明如下：

```
HTTP/1.1 proxies MUST parse the Connection header field before a message is forwarded and, for each connection-token in this field, remove any header field(s) from the message with the same name as the connection-token. Connection options are signaled by the presence of a connection-token in the Connection header field, not by any corresponding additional header field(s), since the additional header field may not be sent if there are no parameters associated with that connection option.
```
综合[RFC2626 14.10][1]、《HTTP权威指南》4.3.1、《图解HTTP》6.3.2中的说法，均指明了 Connection 头部（请求头、响应头）主要包括如下两方面的作用：
1. 控制不再转发给代理的首部字段
2. 管理持久连接

其中，我个人经常见到的是第二种用法，对第一种用法还不甚了解。

第一种用法，大概意思就是，在一个 HTTP 应用（客户端、服务器）将报文转发出之前，必须删除 Connection 首部列出的所有首部字段（多个不同的字段用逗号分隔），当然一些 end-to-end 的头部是肯定不能放进去的，例如 Cache-Control 头。


<!--more-->


### 2.Nginx 是如何做的

注:以下代码均来自 Nginx-1.8.0

对于请求中的 Connection 头， Nginx 在 http 解析时调用 ngx_http_process_connection 方法：

```
static ngx_int_t
ngx_http_process_connection(ngx_http_request_t *r, ngx_table_elt_t *h,
    ngx_uint_t offset)
{
    if (ngx_strcasestrn(h->value.data, "close", 5 - 1)) {
        r->headers_in.connection_type = NGX_HTTP_CONNECTION_CLOSE;

    } else if (ngx_strcasestrn(h->value.data, "keep-alive", 10 - 1)) {
        r->headers_in.connection_type = NGX_HTTP_CONNECTION_KEEP_ALIVE;
    }

    return NGX_OK;
}
```

可见，该方法主要功能只涉及到上面所述的 Connection 作用的第2个作用，而没有第1个。这应该算的上是实现对标准的支持不完整吧。

不管怎么样，我们继续分析一下 Nginx 在管理持久连接上具体是怎么做的，上面的代码逻辑很简单：
（1）如果 Connection 头是 close（不区分大小写），则 r->headers_in.connection_type 被置为 NGX_HTTP_CONNECTION_CLOSE
（2）如果 Connection 头是 keep-alive（不区分大小写），则 r->headers_in.connection_type 被设置为 NGX_HTTP_CONNECTION_KEEP_ALIVE
（3）如果不是以上两种情况，则什么都不做，此时 r->headers_in.connection_type 默认为0.

可以看到，ngx_http_process_connection 的作用仅仅是设置了一个标记r->headers_in.connection_type，我们继续看这个标记是如何被使用的。

要想完整的讲述这个过程，不可避免需要涉及一些 Nginx 配置解析、解析 HTTP 协议相关的流程，但这些都不是本文的重点，下面均一笔带过。

Nginx 的各历史版本中，对于 http 协议解析的过程，细节稍微有些变化。在 Nginx-1.8.0 中，过程如下：
（0）我们知道，Nginx 是一个主进程加多子进程的架构，配置解析发生在 fork 子进程之前，在这个过程中很多操作都是对于全局变量 cycle 的操作。在 fork 之后，每个子进程均会继承一份这个全局变量 cycle(这里不考虑COW)。
（1）http 配置解析的入口为 ngx_http_block，这是个很复杂的函数。因为 Nginx 的配置文件分层级，可以有包含关系，为了对这个特性提供支持，配置文件的解析涉及到配置项的内存布局的设计，最终落实下来，就是全局变量 cycle 的 conf_ctx 成员，这是个四重指针。配置解析的这部分是另一个话题，本文不打算细说。这里要关注的是，在 ngx_http_block 函数的最后，调用了ngx_http_optimize_servers 方法，在这个方法里，完成了这样一个事情：配置文件里的监听套接字（可能是多个），最终被复制到了全局变量 ngx_cycle 的 listening 数组里。整个调用关系为，nginx.c->main()->ngx_init_cycle->ngx_http_block(在 ngx_init_cycle里通过钩子被回调)->ngx_http_optimize_servers->ngx_http_init_listening->ngx_http_add_listening->ngx_create_listening, 有兴趣的同学可以深入进去看看。
（2）在上面的这个调用链里，ngx_http_add_listening 做了另外一个事情：将所有的ngx_listening_t 类型的监听套接字的 handler 钩子设置为 ngx_http_init_connection，这个在后面会用到。
（3）nginx fork 出多个子进程，每个子进程会在 ngx_worker_process_init 方法里调用各个nginx 模块 init_process 钩子，其中当然也包括 NGX_EVENT_MODULE 类型的ngx_event_core_module 模块，其 init_process 钩子为 ngx_event_process_init。在ngx_event_process_init 里，每一个 ngx_listening_t 类型的监听套接字变量 ls[i]，根据ngx_get_connection 从 nginx 的 connections 储备池里获得一个与之相关的ngx_connection_t 类型的变量 c ,这两个变量均有一个指针成员指向对方，以此保持互相联系。从这里开始，我们将注意力转移到这个 ngx_connection_t 类型的变量 c 上。在 ngx_event_process_init 的后面，这个 ngx_connection_t 类型的变量的读事件的 handler，被置为 ngx_event_accept。然后这个读事件被添加到 epoll 中。
（4）当一个请求来临时，ngx_event_accpet 被回调，其中上面第（2）步里为监听套接字设置的 handler,即 ngx_http_init_connection 被调用：

```
ls->handler(c);
```
这个调用非常有趣，ls 实质上是代表着监听套接字，而参数 c 则是 accept 后建立起来的连接套接字，根据 socket 的基本知识，该连接上后续客户端与 Nginx 之间的信息传输，都通过这个连接套接字上的读写来进行。
（5）从 ngx_http_init_connection 开始，就着手进行一系列 HTTP 协议解析。中间涉及到ngx_http_wait_request_handler、ngx_http_create_request、ngx_http_process_request_line、ngx_http_process_request_headers 等解析方法。

似乎已经偏题太远了，我们回到最初的 Connection 请求头，在上面所讲的ngx_http_process_connection 中，根据 Connection 头，将 r->headers_in.connection_type 置为NGX_HTTP_CONNECTION_CLOSE、NGX_HTTP_CONNECTION_KEEP_ALIVE 或者默认初始值 0.

那么这个过程发生在上面所述的一系列流程的哪个阶段呢？当然是发生在ngx_http_process_request_headers 里了，在个方法里，全局数组 ngx_http_headers_in 的对应元素的 handler 被调用，对于 Connection 请求头，就是 ngx_http_process_connection了。

讲明白了 ngx_http_process_connection 发生的前因，我们再来看看后果。

ngx_http_process_connection 的直接影响只有一个，即对 r->headers_in.connection_type进行赋值，close 为 NGX_HTTP_CONNECTION_CLOSE，keep-alive 为NGX_HTTP_CONNECTION_KEEP_ALIVE，其它情况为 0（初始值）。

（6）在第（5）步里，在 ngx_http_process_request_line方法调用完 ngx_http_process_request_headers 后，继续调用 ngx_http_process_request，进而开始正式的在业务上处理HTTP请求。ngx_http_process_request的核心内容是对 ngx_http_handler 的调用。

（7）在 ngx_http_handler 里，有这样一段代码：

```
if (!r->internal) {
        switch (r->headers_in.connection_type) {
        case 0:
            r->keepalive = (r->http_version > NGX_HTTP_VERSION_10);
            break;

        case NGX_HTTP_CONNECTION_CLOSE:
            r->keepalive = 0;
            break;

        case NGX_HTTP_CONNECTION_KEEP_ALIVE:
            r->keepalive = 1;
            break;
        }

        r->lingering_close = (r->headers_in.content_length_n > 0
                              || r->headers_in.chunked);
        r->phase_handler = 0;

    } else {
        cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
        r->phase_handler = cmcf->phase_engine.server_rewrite_index;
    }
```
所以，到现在为止，客户端请求里带来的 Connection 头部，落在了 r->keepalive 上了。规则如下：

* 如果 Connection 头部里为"close",则 r->keepalive 为 0.
* 如果 Connection 头部里为"keep-alive",则 r->keepalive为 1.
* 如果不是以上两种情况，则按照 HTTP 协议走默认情况：如果是 HTTP 1.0 以上，则 r->keepalive 默认为 1，如果是 HTTP 1.0及以下，则 r->keepalive默认为 0.这一点是与 HTTP 协议相符合的.

到目前为止，我们所讲的都算是 r->keepalive 是怎么产生的，还没有涉及到它是如何被使用的。

r->keepalive 的使用主要是在函数 ngx_http_finalize_connection 中，而ngx_http_finalize_connection 在 Nginx 中，仅被 ngx_http_finalize_request 调用。顾名思义，ngx_http_finalize_request 讲的是怎么结束一个 HTTP 请求的。

（8）在 ngx_http_finalize_connection 中，如果 r->keepalive 为1，则会调用ngx_http_set_keepalive 并返回。ngx_http_set_keepalive 方法完成将当前连接设为keepalive 状态的实质性工作。它实际上会把表示请求的 ngx_http_request_t 结构体释放，却又不会调用 ngx_http_close_connection 方法关闭连接，同时也在检测 keepalive 连接是否超时。

至此，Connection 头部在 Nginx 里的处理流程差不多都讲完了，具体细节可按照这个脉络查看相应的代码。


  [1]: https://tools.ietf.org/html/rfc2616#page-117