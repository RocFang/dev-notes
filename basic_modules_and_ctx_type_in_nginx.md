# 1.Nginx中的基本模块类型
Nginx中的模块对外看来主要包含如下几个类别：

1. Core模块
2. HTTP模块（标准HTTP模块和第三方HTTP模块）
3. Mail模块

其中，Core模块是最基本的模块。由于Nginx主要是一个HTTP服务器，所以默认编译时会把HTTP模块给编译进去，而不会编译Mail模块。如果要排除HTTP模块，则在编译时加入选项--without-http,同理，如果要包含Mail模块，则在编译时加入选项--with-mail.
 
# 2.Nginx的基本模块
我们先看Nginx有哪些基本模块，将HTTP模块排除在外。执行如下操作:
```
cd nginx-1.6.2
./configure --without-http
```
此时，会生成objs目录，打开objs/ngx_modules.c，可以看到其内容为:
```
#include <ngx_config.h>
#include <ngx_core.h>



extern ngx_module_t  ngx_core_module;
extern ngx_module_t  ngx_errlog_module;
extern ngx_module_t  ngx_conf_module;
extern ngx_module_t  ngx_events_module;
extern ngx_module_t  ngx_event_core_module;
extern ngx_module_t  ngx_epoll_module;

ngx_module_t *ngx_modules[] = {
    &ngx_core_module,
    &ngx_errlog_module,
    &ngx_conf_module,
    &ngx_events_module,
    &ngx_event_core_module,
    &ngx_epoll_module,
    NULL
};
```
其中，ngx_modules数组中包含的几个模块即为Nginx中的几个基本模块。

1. ngx_core_module
2. ngx_errlog_module
3. ngx_conf_module
4. ngx_events_module
5. ngx_event_core_module
6. ngx_epoll_module

# 3.Nginx模块的基本数据结构
每一个Nginx模块，都是一个ngx_module_t类型的变量，ngx_module_t类型定义如下:
```
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;
    ngx_uint_t            spare2;
    ngx_uint_t            spare3;

    ngx_uint_t            version;

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
```
例如，http模块的定义为:
```
ngx_module_t  ngx_http_module = {
    NGX_MODULE_V1,
    &ngx_http_module_ctx,                  /* module context */
    ngx_http_commands,                     /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```

从上述结构体中，我们可以看到一个ctx成员。ctx成员是Nginx模块的重要组成部分。基本上，每一个Nginx模块（我们一般说一个Nginx模块，指的就是一个ngx_module_t类型的变量）都会定义一个ctx变量，之所以说基本上，是因为也有例外。例如ngx_conf_module的定义为:
```
ngx_module_t  ngx_conf_module = {
    NGX_MODULE_V1,
    NULL,                                  /* module context */
    ngx_conf_commands,                     /* module directives */
    NGX_CONF_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    ngx_conf_flush_files,                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```
可以看到，其ctx成员为NULL。
在ngx_module_t的类型定义中，我们看到ctx是一个void*类型的指针，意味着其类型可能不是唯一的。
那么Nginx中有几种类型的ctx呢？

# 4.Nginx中的几个基本的ctx类型
Nginx中目前主要包含如下几个ctx类型：

1. ngx_core_module_t
2. ngx_event_module_t
3. ngx_http_module_t
4. ngx_mail_module_t
5. 其他

## 4.1 ngx_core_module_t
ngx_core_module_t的定义为:
```
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```
Nginx-1.6.2中，ngx_core_module_t类型的模块（即ctx成员为ngx_core_module_t类型的ngx_module_t变量)包括：

1. ngx_core_module
2. ngx_events_module
3. ngx_openssl_module
4. ngx_google_perftools_module
5. ngx_http_module
6. ngx_errlog_module
7. ngx_mail_module
8. ngx_regex_module

这些模块的type成员均为常量NGX_CORE_MODULE，在遍历ngx_modules数组时，经常用该变量作为ngx_core_module_t类型模块的标记。

## 4.2 ngx_event_module_t
ngx_event_module_t的定义为:
```
typedef struct {
    ngx_str_t              *name;

    void                 *(*create_conf)(ngx_cycle_t *cycle);
    char                 *(*init_conf)(ngx_cycle_t *cycle, void *conf);

    ngx_event_actions_t     actions;
} ngx_event_module_t;

typedef struct {
    ngx_int_t  (*add)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*del)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*enable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);
    ngx_int_t  (*disable)(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags);

    ngx_int_t  (*add_conn)(ngx_connection_t *c);
    ngx_int_t  (*del_conn)(ngx_connection_t *c, ngx_uint_t flags);

    ngx_int_t  (*process_changes)(ngx_cycle_t *cycle, ngx_uint_t nowait);
    ngx_int_t  (*process_events)(ngx_cycle_t *cycle, ngx_msec_t timer,
                   ngx_uint_t flags);

    ngx_int_t  (*init)(ngx_cycle_t *cycle, ngx_msec_t timer);
    void       (*done)(ngx_cycle_t *cycle);
} ngx_event_actions_t;
```
Nginx-1.6.2中，ngx_event_module_t类型的模块（即ctx成员为ngx_event_module_t类型的ngx_module_t变量)包括：

1. ngx_aio_module
2. ngx_devpoll_module
3. ngx_epoll_module
4. ngx_event_core_module
5. ngx_eventport_module
6. ngx_kqueue_module
7. ngx_poll_module
8. ngx_rtsig_module
9. ngx_select_module

一般来说，在Linux上，需要关注的是ngx_epoll_module和ngx_event_core_module.

这些模块的type成员均为常量NGX_EVENT_MODULE，在遍历ngx_modules数组时，经常用该变量作为ngx_event_module_t类型模块的标记。

## 4.3 ngx_http_module_t
ngx_http_module_t的定义为：
```
typedef struct {
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    void       *(*create_main_conf)(ngx_conf_t *cf);
    char       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void       *(*create_srv_conf)(ngx_conf_t *cf);
    char       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    void       *(*create_loc_conf)(ngx_conf_t *cf);
    char       *(*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```
ngx_http_module_t类型的模块就太多了，我们通常说的Nginx模块编程，绝大部分即为此类。

Nginx-1.6.2中，原生的ngx_http_module_t类型的模块（即ctx为ngx_http_module_t类型的ngx_module_t变量)包括:

1. ngx_http_access_module
2. ngx_http_addition_filter_module
3. ngx_http_auth_basic_module
4. ngx_http_auth_request_module
5. ngx_http_autoindex_module
6. ngx_http_browser_module
7. ngx_http_charset_filter_module
8. ngx_http_chunked_filter_module
9. ngx_http_copy_filter_module
10. ngx_http_core_module
11. ngx_http_dav_module
12. ngx_http_degradation_module
13. ngx_http_empty_gif_module
14. ngx_http_fastcgi_module
15. ngx_http_flv_module
16. ngx_http_geoip_module
17. ngx_http_geo_module
18. ngx_http_gunzip_filter_module
19. ngx_http_gzip_filter_module
20. ngx_http_gzip_static_module
21. ngx_http_headers_filter_module
22. ngx_http_header_filter_module
23. ngx_http_image_filter_module
24. ngx_http_index_module
25. ngx_http_limit_conn_module
26. ngx_http_limit_req_module
27. ngx_http_log_module
28. ngx_http_map_module
29. ngx_http_memcached_module
30. ngx_http_mp4_module
31. ngx_http_not_modified_filter_module
32. ngx_http_perl_module
33. ngx_http_postpone_filter_module
34. ngx_http_proxy_module
35. ngx_http_random_index_module
36. ngx_http_range_header_filter_module
37. ngx_http_range_body_filter_module
38. ngx_http_realip_module
39. ngx_http_referer_module
40. ngx_http_rewrite_module
41. ngx_http_scgi_module
42. ngx_http_secure_link_module
43. ngx_http_spdy_filter_module
44. ngx_http_spdy_module
45. ngx_http_split_clients_module
46. ngx_http_ssi_filter_module
47. ngx_http_ssl_module
48. ngx_http_static_module
49. ngx_http_stub_status_module
50. ngx_http_sub_filter_module
51. ngx_http_upstream_module
52. ngx_http_upstream_ip_hash_module
53. ngx_http_upstream_keepalive_module
54. ngx_http_upstream_least_conn_module
55. ngx_http_userid_filter_module
56. ngx_http_uwsgi_module
57. ngx_http_write_filter_module
58. ngx_http_xslt_filter_module

这些模块的type成员均为常量NGX_HTTP_MODULE，在遍历ngx_modules数组时，经常用该变量作为ngx_http_module_t类型模块的标记。

## 4.4 ngx_mail_module_t
ngx_mail_module_t的定义为:
```
typedef struct {
    ngx_mail_protocol_t        *protocol;

    void                       *(*create_main_conf)(ngx_conf_t *cf);
    char                       *(*init_main_conf)(ngx_conf_t *cf, void *conf);

    void                       *(*create_srv_conf)(ngx_conf_t *cf);
    char                       *(*merge_srv_conf)(ngx_conf_t *cf, void *prev,
                                      void *conf);
} ngx_mail_module_t;
```
Nginx-1.6.2中，ngx_mail_module_t类型的模块（即ctx成员为ngx_mail_module_t类型的ngx_module_t变量)包括：

1. ngx_mail_auth_http_module
2. ngx_mail_core_module
3. ngx_mail_imap_module
4. ngx_mail_pop3_module
5. ngx_mail_proxy_module
6. ngx_mail_smtp_module
7. ngx_mail_ssl_module

这些模块的type成员均为常量NGX_MAIL_MODULE，在遍历ngx_modules数组时，经常用该变量作为ngx_mail_module_t类型模块的标记。

## 4.5其他模块
这里，主要是ngx_conf_module,其定义为:
```
ngx_module_t  ngx_conf_module = {
    NGX_MODULE_V1,
    NULL,                                  /* module context */
    ngx_conf_commands,                     /* module directives */
    NGX_CONF_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    ngx_conf_flush_files,                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};
```
该模块有如下两点值得注意:

1. 如前所述，其ctx成员为NULL.
2. 其type为NGX_CONF_MODULE
