# Nginx-1.9.8推出的切片模块

熟悉 CDN 行业主流技术的朋友应该都比较清楚，虽然 Nginx 近几年发展的如日中天，但是基本上没有直接使用它自带的 proxy_cache 模块来做缓存的，原因有很多，例如下面几个：

* 不支持多盘
* 不支持裸设备
* 大文件不会切片
* 大文件的 Range 请求表现不尽如人意
* Nginx 自身不支持合并回源


<!--more-->


当然，上面列出来的并不全， 所以，在现在主流的 CDN 技术栈里面， Nginx 起到的多是一个粘合剂的作用，例如调度器、负载均衡器、业务逻辑（防盗链等），需要与 Squid、ATS 等主流 Cache Server 配合使用，即使不使用这些技术，也会使用其它办法，例如直接使用文件系统和数据库去管理文件，而不会直接使用 proxy_cache 模块。

Nginx-1.9.8 中新增加的一个模块ngx_http_slice_module解决了一部分问题。本文就来尝尝鲜，看看这个切片模块。

注意：截至到发文时，Nginx 马上发布了 Nginx-1.9.9，用来解决 Nginx-1.9.8中的一个 Bug，所以，在实际使用中，如果需要使用本新增特性，请直接使用 Nginx-1.9.9。

首先，我们看看几个不同版本的 Nginx 的 proxy_cache 对 Range 的处理情况。

### Nginx-0.8.15

在 Nginx-0.8.15 中，使用如下配置文件做测试:

```
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    proxy_cache_path /tmp/nginx/cache levels=1:2 keys_zone=cache:100m;
    server {    
        listen       8087;
        server_name  localhost;
        location / {
            proxy_cache cache;
            proxy_cache_valid 200 206 1h;
           # proxy_set_header Range $http_range;
            proxy_pass http://127.0.0.1:8080;

        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
}
```
重点说明以下两种情况：
* 第一次 Range 请求（没有本地缓存），Nginx 会去后端将整个文件拉取下来（后端响应码是200）后，并且返回给客户端的是整个文件，响应状态码是200，而非206. 后续的 Range 请求则都用缓存下来的本地文件提供服务，且响应状态码都是对应的206了。
* 如果在上面的配置文件中，加上 proxy_set_header Range $http_range;再进行测试(测试前先清空 Nginx 本地缓存)。则第一次 Range 请求（没有本地缓存），Nginx 会去后端用 Range 请求文件，而不会把整个文件拉下来，响应给客户端的也是206.但问题在于，由于没有把 Range 请求加入到 cache key 中，会导致后续所有的请求，不管 Range 如何，只要 url 不变，都会直接用cache 的内容来返回给客户端，这肯定是不符合要求的。

### Nginx-1.9.7

在 Nginx-1.9.7 中，同样进行上面两种情况的测试，第二种情况的结果其实是没多少意义，而且肯定也和 Nginx-0.8.15 一样，所以这里只关注第一种测试情况。

第一次 Range 请求（没有本地缓存），Nginx 会去后端将整个文件拉取下来（后端响应码是200），但返回给客户端的是正确的 Range 响应，即206.后续的 Range 请求，则都用缓存下来的本地文件提供服务，且都是正常的206响应。

可见，与之前的版本相比，还是有改进的，但并没有解决最实质的问题。

我们可以看看 Nginx 官方对于 Cache 在 Range 请求时行为的说明:
>How Does NGINX Handle Byte Range Requests?
>
>If the file is up-to-date in the cache, then NGINX honors a byte range request and serves only the specified bytes of the item to the client. If the file is not cached, or if it’s stale, NGINX downloads the entire file from the origin server. If the request is for a single byte range, NGINX sends that range to the client as soon as it is encountered in the download stream. If the request specifies multiple byte ranges within the same file, NGINX delivers the entire file to the client when the download completes.
>
>Once the download completes, NGINX moves the entire resource into the cache so that all future byte-range requests, whether for a single range or multiple ranges, are satisfied immediately from the cache.

### Nginx-1.9.8

我们继续看看Nginx-1.9.8, 当然，在编译时要加上参数--with-http_slice_module，并作类似下面的配置:

```
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    proxy_cache_path /tmp/nginx/cache levels=1:2 keys_zone=cache:100m;
    server {
        listen       8087;
        server_name  localhost;
        location / {
            slice 1m;
            proxy_cache cache;
            proxy_cache_key $uri$is_args$args$slice_range;
            proxy_set_header Range $slice_range;
            proxy_cache_valid 200 206 1h;
            #proxy_set_header Range $http_range;
            proxy_pass http://127.0.0.1:8080;

        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
}
```
不测不知道，一侧吓一跳，这俨然是一个杀手级的特性。

首先，如果不带 Range 请求，后端大文件在本地 cache 时，会按照配置的 slice 大小进行切片存储。

其次，如果带 Range 请求，则 Nginx 会用合适的 Range 大小（以 slice 为边界）去后端请求，这个大小跟客户端请求的 Range 可能不一样，并将以 slice 为大小的切片存储到本地，并以正确的206响应客户端。

注意上面所说的，Nginx 到后端的 Range 并不一定等于客户端请求的 Range，因为无论你请求的Range 如何，Nginx 到后端总是以 slice 大小为边界，将客户端请求分割成若干个子请求到后端，假设配置的 slice 大小为1M,即1024字节，那么如果客户端请求 Range 为0-1023范围以内任何数字，均会落到第一个切片上，如果请求的 Range 横跨了几个 slice 大小，则nginx会向后端发起多个子请求，将这几个 slice 缓存下来。而对客户端，均以客户端请求的 Range 为准。如果一个请求中，有一部分文件之前没有缓存下来，则 Nginx 只会去向后端请求缺失的那些切片。

由于这个模块是建立在子请求的基础上，会有这么一个潜在问题：当文件很大或者 slice 很小的时候，会按照 slice 大小分成很多个子请求，而这些个子请求并不会马上释放自己的资源，可能会导致文件描述符耗尽等情况。

### 小结

总结一下，需要注意的点：

* 该模块用在 proxy_cache 大文件的场景，将大文件切片缓存
* 编译时对 configure 加上 --with-http_slice_module 参数
* $slice_range 一定要加到 proxy_cache_key 中，并使用 proxy_set_header 将其作为 Range 头传递给后端
* 要根据文件大小合理设置 slice 大小

具体特性的说明，可以参考 Roman Arutyunyan 提出这个 patch 时的邮件来往：
[https://forum.nginx.org/read.php?29,261929,261929#msg-261929][1]

顺带提一下，Roman Arutyunyan 也是个大牛，做流媒体领域的同学们肯定很多都听说过：[nginx-rtmp][2] 模块的作者。

### 参考资料

1. Nginx 官方的 Cache 指南
[https://www.nginx.com/blog/nginx-caching-guide/][3]
2. Nginx各版本changelog
[http://nginx.org/en/CHANGES][4]
3. Nginx proxy 模块 wiki
[http://nginx.org/en/docs/http/ngx_http_proxy_module.html][5]
4. http_slice_module 的历次提交记录
[http://hg.nginx.org/nginx/rev/29f35e60840b][6]
[http://hg.nginx.org/nginx/rev/bc9ea464e354][7]
[http://hg.nginx.org/nginx/rev/4f0f4f02c98f][8]
5. http_slice_module 提交前的邮件来往
[https://forum.nginx.org/read.php?29,261929][9]
6. Nginx 之前版本关于 Range cache 的邮件来往
[https://forum.nginx.org/read.php?2,8958,8958][10]
7. 切片模块的 wiki
[http://nginx.org/en/docs/http/ngx_http_slice_module.html][11]


  [1]: https://forum.nginx.org/read.php?29,261929,261929#msg-261929
  [2]: https://github.com/arut/nginx-rtmp-module
  [3]: https://www.nginx.com/blog/nginx-caching-guide/
  [4]: http://nginx.org/en/CHANGES
  [5]: http://nginx.org/en/docs/http/ngx_http_proxy_module.html
  [6]: http://hg.nginx.org/nginx/rev/29f35e60840b
  [7]: http://hg.nginx.org/nginx/rev/bc9ea464e354
  [8]: http://hg.nginx.org/nginx/rev/4f0f4f02c98f
  [9]: https://forum.nginx.org/read.php?29,261929
  [10]: https://forum.nginx.org/read.php?2,8958,8958
  [11]: http://nginx.org/en/docs/http/ngx_http_slice_module.html