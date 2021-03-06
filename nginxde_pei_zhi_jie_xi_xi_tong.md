# Nginx的配置解析系统

## 几个重要的指令类型属性

我们知道，每一个Nginx的模块就是对应一个ngx_module_t类型的结构体，该结构体的commands成员是一个数组，里面包括了该模块的配置指令。每一个commands成员是ngx_command_t类型的结构体，其定义如下:
```
struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```
我们重点关注这个type成员。有如下几个比较重要的type属性需要着重熟悉:
* NGX_MAIN_CONF
* NGX_DIRECT_CONF
* NGX_CONF_BLOCK

搜索一下nginx-1.6.2的源代码，发现，type值包含了NGX\_DIRECT\_CONF的指令所在的模块分别是:
* ngx\_core\_module模块的所有指令。
* ngx\_openssl\_module模块的所有指令。其实只有一个ssl\_engine
* ngx\_google\_perftools\_module模块的所有指令。其实只有一个google\_perftools\_profiles
* ngx\_regex\_module模块的所有指令。其实只有一个pcre\_jit

这几个模块有如下共同点:
* 模块类型都是NGX\_CORE\_MODULE
* 指令类型NGX\_DIRECT\_CONF都是与NGX\_MAIN\_CONF共同出现

在[Nginx中的几个基本模块和几个模块ctx类型](nginxzhong_de_ji_ge_ji_ben_mo_kuai_he_ji_ge_mo_kuai_ctx_lei_xing.md)中，我们讲过，Nginx-1.6.2中，ngx_core_module_t类型的模块（即ctx成员为ngx_core_module_t类型的ngx_module_t变量，也即type成员为NGX\_CORE\_MODULE的ngx_module_t变量)包括：

1. ngx_core_module
2. ngx_events_module
3. ngx_openssl_module
4. ngx_google_perftools_module
5. ngx_http_module
6. ngx_errlog_module
7. ngx_mail_module
8. ngx_regex_module

这些NGX\_CORE\_MODULE类型的模块中，还有如下几个，其配置指令集中没有NGX\_DIRECT\_CONF类型的指令:

1. ngx_events_module
2. ngx_http_module
3. ngx_errlog_module
4. ngx_mail_module

其中：

1. ngx_events_module模块只有一个指令"events",是一个配置块类型的指令，即指令类型包含NGX\_CONF\_BLOCK.
2. ngx_http_module模块只有一个指令"http",同样是一个配置块类型的指令,即指令类型包含NGX\_CONF\_BLOCK.
3. ngx_errlog_module
4. ngx_mail_module模块有两个指令“mail”和“imap”，且都是配置块类型的指令，即指令类型包括NGX\_CONF\_BLOCK.

搜索nginx-1.6.2代码，发现type值包含了NGX\_MAIN\_CONF的指令所在的模块
为:
* ngx\_core\_module模块的所有指令。
* ngx_events_module模块的所有指令。其实只有一个“events”
* ngx_openssl_module模块的所有指令。其实只有一个“ssl_engine”
* ngx_google_perftools_module模块的所有指令。其实只有一个"google_perftools_profiles"
* ngx_http_module模块的所有指令，其实只有一个"http".
* ngx_errlog_module模块的所有指令，其实只有一个"error_log"
* ngx_mail_module模块的所有指令，包含"mail"和"imap"
* ngx_regex_module模块的所有指令，包含"pcre_jit"

这几个模块有如下共同点:
* 模块类型都是NGX_CORE_MODULE

所以，综上，在nginx-1.6.2中，关于NGX_MAIN_CONF和NGX_DIRECT_CONF有如下几点:
* NGX_MAIN_CONF和NGX_DIRECT_CONF都只会出现在NGX_CORE_MODULE类型的模块中。
* 所有的NGX_CORE_MODULE类型的模块，其指令均有NGX_MAIN_CONF属性
* 所有NGX_CORE_MODULE类型的模块，除了**ngx\_errlog\_module**模块外,其指令如果没有NGX_DIRECT_CONF属性，则会有NGX_CONF_BLOCK属性。ngx_errlog_module模块比较特殊，其指令属性为“NGX_MAIN_CONF|NGX_CONF_1MORE”
* 所有的nginx模块中，NGX_DIRECT_CONF和NGX_CONF_BLOCK属性不会同时出现。

另外，type值包含了NGX_CONF_BLOCK属性的指令所在的模块有:

* ngx_events_module模块，指令只有“events”
* ngx_http_module模块，指令只有“http”
* ngx_http_charset_filter_module模块的“charset_map”指令
* ngx_http_core_module模块的server指令、location指令、types指令、limit_except指令
* ngx_http_geo_module模块的geo指令
* ngx_http_map_module模块的map指令
* ngx_http_rewrite_module模块的if指令
* ngx_http_split_clients_module模块的split_clients指令
* ngx_http_upstream_module模块的upstream指令
* ngx_mail_module模块的mail指令和imap指令
* ngx_mail_core_module模块的server指令

列出这些不是为了去死记硬背。而是我们可以很直观的了解到,NGX_CONF_BLOCK属性只能表示该命令后面跟的是一个复杂配置项而已。

## Nginx配置解析的流程

Nginx配置解析的流程如下图所示:
![](nginx-config-process.jpg)


参考文章http://tech.uc.cn/?p=300







