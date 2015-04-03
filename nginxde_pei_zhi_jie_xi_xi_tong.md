# Nginx的配置解析系统

## 配置指令的类型

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
我们重点关注这个type成员。
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

搜索nginx-1.6.2代码，发现type值包含了NGX\_MAIN\_CONF的指令所在的模块为:
* ngx\_core\_module模块的所有指令。
* ngx_events_module模块的所有指令。其实只有一个“events”
* ngx_openssl_module模块的所有指令。其实只有一个“ssl_engine”
* 





