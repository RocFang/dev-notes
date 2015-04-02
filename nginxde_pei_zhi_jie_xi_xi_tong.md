# Nginx的配置解析系统

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
搜索一下nginx-1.6.2的源代码，发现，type值为NGX\_DIRECT\_CONF的指令所在的模块分别是:
* ngx\_core\_module模块的所有指令。
* ngx\_openssl\_module模块的所有指令。其实只有一个ssl\_engine
* ngx\_google\_perftools\_module模块的所有指令。其实只有一个google\_perftools\_profiles
* ngx\_regex\_module模块的所有指令。其实只有一个pcre\_jit

这几个模块有如下共同点:
* 模块类型都是NGX\_CORE\_MODULE
* 指令类型NGX\_DIRECT\_CONF都是与NGX\_MAIN\_CONF共同出现

