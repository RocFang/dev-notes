# ngx.var.arg与ngx.req.get_uri_args的区别

ngx.var.arg\_xx与ngx.req.get\_uri\_args["xx"]两者都是为了获取请求uri中的参数，例如
```
http://pureage.info?strider=1
```

为了获取输入参数strider，以下两种方法都可以：

1. local strider = ngx.var.arg_strider
2. local strider = ngx.req.get_uri_args["strider"]

差别在于，当请求uri中有多个同名参数时,ngx.var.arg\_xx的做法是取第一个出现的值,ngx.req\_get_uri_args["xx"]的做法是返回一个table，该table里存放了该参数的所有值，例如,当请求uri为：
```
http://pureage.info?strider=1&strider=2&strider=3&strider=4
```
时，ngx.var.arg\_strider的值为"1",而ngx.req.get\_uri\_args["strider"]的值为table ["1", "2", "3", "4"]。因此，ngx.req.get\_uri\_args属于ngx.var.arg_的增强。

ngx.var.arg\_的实现是直接使用nginx原生的变量支持，nginx相关代码为：

```
ngx_http_variable_value_t *
ngx_http_get_variable(ngx_http_request_t *r, ngx_str_t *name, ngx_uint_t key)
{
....(略）....
if (ngx_strncmp(name->data, "arg_", 4) == 0) {

        if (ngx_http_variable_argument(r, vv, (uintptr_t) name) == NGX_OK) {
            return vv;
        }

        return NULL;
    }

    vv->not_found = 1;

    return vv;
}

static ngx_int_t
ngx_http_variable_argument(ngx_http_request_t *r, ngx_http_variable_value_t *v,
    uintptr_t data)
{
    ngx_str_t *name = (ngx_str_t *) data;

    u_char *arg;
    size_t len;
    ngx_str_t value;

    len = name->len - (sizeof("arg_") - 1);
    arg = name->data + sizeof("arg_") - 1;

    if (ngx_http_arg(r, arg, len, &value) != NGX_OK) {
        v->not_found = 1;
        return NGX_OK;
    }

    v->data = value.data;
    v->len = value.len;
    v->valid = 1;
    v->no_cacheable = 0;
    v->not_found = 0;

    return NGX_OK;
}

ngx_int_t
ngx_http_arg(ngx_http_request_t *r, u_char *name, size_t len, ngx_str_t *value)
{
    u_char *p, *last;

    if (r->args.len == 0) {
        return NGX_DECLINED;
    }

    p = r->args.data;
    last = p + r->args.len;

    for ( /* void */ ; p < last; p++) {

        /* we need '=' after name, so drop one char from last */

        p = ngx_strlcasestrn(p, last - 1, name, len - 1);

        if (p == NULL) {
            return NGX_DECLINED;
        }

        if ((p == r->args.data || *(p - 1) == '&') && *(p + len) == '=') {

            value->data = p + len + 1;

            p = ngx_strlchr(p, last, '&');

            if (p == NULL) {
                p = r->args.data + r->args.len;
            }

            value->len = p - value->data;

            return NGX_OK;
        }
    }

    return NGX_DECLINED;
}
```



