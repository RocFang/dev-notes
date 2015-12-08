# nginx proxy_cache与etag配合的问题

**首先谈谈遇到的问题:**

一个Nginx架在一个后端服务的前面，Nginx proxy_pass到它并开启proxy_cache,假设这个后端服务总是会吐Etag响应头。
在应用中，我们发现当nginx 的proxy_cache成功将后端的页面cache住时,浏览器多次对该页面发起请求，会命中nginx的cache,但即使浏览器请求带了If-None-Match请求头，nginx却不会响应304，而是响应200.
这样带来的问题是，即使nginx的cache将请求阻挡在后端应用之外，但是:
（1）命中后每次响应200导致我们nginx所在的服务器和客户浏览器双方都有流量损耗
（2）更重要的是增长了我们的服务响应时间。因为，如果是304的话，nginx不需要向浏览器吐数据，只用告诉浏览器用本地的缓存就好了。


排查过程涉及个版本的测试和代码比对，这里就省略了，下面说下结论:

**一. nginx-1.7.3之前的版本:**

* nginx很早就支持etag，但是nginx-1.7.3之前不支持弱etag。
* nginx-1.7.3之前版本在proxy_cache时，会有上面提到的问题。
* 之所以会遇到上面的问题，是因为nginx里面有很多filter模块，比如ssl,xsl,gzip等等，这些filter模块本质上算是对响应做出了修改。所以，Nginx为了严格遵循etag的本意，这种情况下就认为etag应该失效。而早期版本又不支持弱etag验证，所以干脆就不承认etag，每次都返回200,而不是304.

**二.nginx-1.7.3及其之后的版本:**

首先我们看看nginx-1.7.3的changelog：
```
Changes with nginx 1.7.3                                         08 Jul 2014
    *) Feature: weak entity tags are now preserved on response
       modifications, and strong ones are changed to weak.
    *) Feature: cache revalidation now uses If-None-Match header if
       possible.
    *) Feature: the "ssl_password_file" directive.
    *) Bugfix: the If-None-Match request header line was ignored if there
       was no Last-Modified header in a response returned from cache.
    *) Bugfix: "peer closed connection in SSL handshake" messages were
       logged at "info" level instead of "error" while connecting to
       backends.
    *) Bugfix: in the ngx_http_dav_module module in nginx/Windows.
    *) Bugfix: SPDY connections might be closed prematurely if caching was
       used.
```
**可以看到，主要有两点：**

* 增加了弱etag检验功能：对于那些修改了响应的filter模块，nginx启用弱etag检验。
* cache住的文件，其验证会优先采用If-None-Match校验，即Etag校验。

那么，在nginx-1.7.3及其以后的版本，其Etag功能可以描述如下:

1. Etag功能得到增强，既有强Etag，又有弱Etag.其实，但是所谓的强弱，只是为了遵循标准而分出来的两种说法而已。
2. 对于Nginx本地的静态文件，是强Etag验证。
3. 对于Nginx本地的非静态内容，不做Etag验证。这里可以用nginx-lua模块ngx.say来简单验证。
4. 对于proxy_cache缓存住的文件,无论该文件是后端的静态文件，或是后端动态产生的页面，只要后端吐出了Etag响应头，则Nginx对客户端过来的请求，都会启动Etag校验。即，第一次请求，Nginx会将后端吐出的Etag头传给客户端，客户端后面再请求时，会带上 If-None-Match请求头，如果校验通过，会直接返回304.（这就解决了我们的问题)
5. 注意，上面这一点又可以分为两种情况，第一种是如果Nginx在将cache住的内容吐给浏览器时，如果Nginx不启用filter模块来修改响应，则Etag的强弱跟后端传过来的相同。第二种情况，如果Nginx在将cache住的内容吐给浏览器时，如果启用了filter模块，即响应头或体被修改，那么Nginx会将后端的强Etag转换为弱Etag.例如：如果后端本来返回的Etag为ETag: "12345",则Nginx会将其弱化，吐给浏览器，即改为Etag: W/"12345", 浏览器下次请求的If-None-Match请求头也变为If-None-Match: W/"12345"。