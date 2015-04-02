# 用proxy_intercept_errors和recursive_error_pages代理多次302

302是HTTP协议中的一个经常被使用状态码，是多种重定向方式的一种，其语义经常被解释为“Moved Temporarily”。这里顺带提一下，现实中用到的302多为误用（与303，307混用），在HTTP/1.1中，它的语义为“Found”.

302有时候很明显，有时候又比较隐蔽。最简单的情况，是当我们在浏览器中输入一个网址A，然后浏览器地址栏会自动跳到B，进而打开一个网页，这种情况就很可能是302。

比较隐蔽的情况经常发生在嵌入到网页的播放器中。例如，当你打开一个优酷视频播放页面时，抓包观察一下就会经常发现302的影子。但由于这些url并不是直接在浏览器中打开的，所以在浏览器的地址栏看不到变化，当然，如果将这些具体的url特意挑出来复制到浏览器地址栏里，还是可以观察到的。

上一段提到了优酷。其实现在多数在线视频网站都会用到302，原因很简单，视频网站流量一般较大，都会用到CDN,区别只在于是用自建CDN还是商业CDN。而由于302的重定向语义（再重复一遍，302的语义广泛的被误用，在使用302的时候，我们很可能应该使用303或307，但后面都不再纠结这一点），可以与CDN中的调度很好的结合起来。

我们来看一个例子，打开一个网易视频播放页面，抓一下包，找到302状态的那个url。例如：
```
http://flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4
```
我们把它复制到浏览器地址栏中，会发现地址栏迅速的变为了另外一个url，这个Url是不定的，有可能为：
```
http://14.18.140.83/f6c00af500000000-1408987545-236096587/data6/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4
```
用curl工具会更清楚的看到整个过程：
```
curl -I "http://flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4" -L
HTTP/1.1 302 Moved Temporarily 
Server: nginx 
Date: Mon, 25 Aug 2014 14:49:43 GMT 
Content-Type: text/html 
Content-Length: 154 
Connection: keep-alive 
NG: CCN-SW-1-5L2 
X-Mod-Name: GSLB/3.1.0 
Location: http://119.134.254.9/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 

HTTP/1.1 302 Moved Temporarily 
Server: nginx 
Date: Mon, 25 Aug 2014 14:49:41 GMT 
Content-Type: text/html 
Content-Length: 154 
Connection: keep-alive 
X-Mod-Name: Mvod-Server/4.3.3 
Location: http://119.134.254.7/cc89fdac00000000-1408983581-2095617481/data4/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 
NG: CHN-SW-1-3Y1 

HTTP/1.1 200 OK 
Server: nginx 
Date: Mon, 25 Aug 2014 14:49:41 GMT 
Content-Type: video/mp4 
Content-Length: 3706468 
Last-Modified: Mon, 25 Aug 2014 00:23:50 GMT 
Connection: keep-alive 
Cache-Control: no-cache 
ETag: "53fa8216-388e64" 
NG: CHN-SW-1-3g6 
X-Mod-Name: Mvod-Server/4.3.3 
Accept-Ranges: bytes 
```
可以看到，这中间经历了两次302。

先暂时将这个例子放在一边，再来说说另一个重要的术语：proxy.我们通常会戏称，某些领导是302类型的，某些领导是proxy类型的。302类型的领导，一件事情经过他的手，会迅速的转给他人，而proxy类型的领导则会参与到事情中来，甚至把事情全部做完。

回到上面的例子，如果访问一个url中途会有多个302，那如果需要用Nginx设计一个proxy，来隐藏掉中间所有的这些302，该怎么做呢

## 1.原始Proxy

我们知道，Nginx本身就是一个优秀的代理服务器。因此，首先我们来架设一个Nginx正向代理，服务器IP为192.168.109.128（我的一个测试虚拟机）。

初始配置简化如下：
```
server {
        listen 80;
        location / {
                rewrite_by_lua '
                        ngx.exec("/proxy-to" .. ngx.var.request_uri)
                ';
        }

        location ~ /proxy-to/([^/]+)(.*) {
                proxy_pass http://$1$2$is_args$query_string;

        }
}
```
实现的功能是，当使用
```
http://192.168.109.128/xxxxxx
```
访问该代理时，会proxy到xxxxxx所代表的真实服务器。

测试结果如下：
```
curl -I "http://192.168.109.128/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4" -L
HTTP/1.1 302 Moved Temporarily 
Server: nginx/1.4.6 
Date: Mon, 25 Aug 2014 14:50:54 GMT 
Content-Type: text/html 
Content-Length: 154 
Connection: keep-alive 
NG: CCN-SW-1-5L2 
X-Mod-Name: GSLB/3.1.0 
Location: http://183.61.140.24/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 

HTTP/1.1 302 Moved Temporarily 
Server: nginx 
Date: Mon, 25 Aug 2014 14:50:55 GMT 
Content-Type: text/html 
Content-Length: 154 
Connection: keep-alive 
X-Mod-Name: Mvod-Server/4.3.3 
Location: http://183.61.140.20/540966e500000000-1408983655-236096587/data1/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 
NG: CHN-ZJ-4-3M4 

HTTP/1.1 200 OK 
Server: nginx 
Date: Mon, 25 Aug 2014 14:50:55 GMT 
Content-Type: video/mp4 
Content-Length: 3706468 
Last-Modified: Mon, 25 Aug 2014 00:31:03 GMT 
Connection: keep-alive 
Cache-Control: no-cache 
ETag: "53fa83c7-388e64" 
NG: CHN-ZJ-4-3M4 
X-Mod-Name: Mvod-Server/4.3.3 
Accept-Ranges: bytes
```

可见，虽然使用proxy，但过程与原始访问没有什么区别。访问过程为，当访问
```
http://192.168.109.128/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4
```
时，Nginx会将该请求proxy到
```
http://flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4
```
而后者马上就会返回一个302，所以Nginx作为proxy，将该302传回到客户端，客户端重新发起请求，进而重复之前的多次302.这里说明一个问题，一旦Nginx的proxy的后端返回302后，客户端即与Nginx这个proxy脱离关系了，Nginx无法起到完整的代理的作用。
## 2. 第1次修改

将配置文件修改为：
```
server {
        listen 80;
        location / {
                rewrite_by_lua '
                        ngx.exec("/proxy-to" .. ngx.var.request_uri)
                ';
        }

        location ~ /proxy-to/([^/]+)(.*) {
                proxy_pass http://$1$2$is_args$query_string;
                error_page 302 = @error_page_302;

        }
        location @error_page_302 {
                rewrite_by_lua '
                        local _, _, upstream_http_location = string.find(ngx.var.upstream_http_location, "^http:/(.*)$")
                        ngx.header["zzzz"] = "/proxy-to" .. upstream_http_location
                        ngx.exec("/proxy-to" .. upstream_http_location);
                ';

        }
}
```
与上面的区别在于，使用了一个error_page，目的是当发现proxy的后端返回302时，则用这个302的目的location继续proxy，而不是直接返回给客户端。并且这个逻辑里面包含着递归的意思，一路跟踪302，直到最终返回200的那个地址。测试结果如下：
```
curl -I "http://192.168.109.128/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4" -L
HTTP/1.1 302 Moved Temporarily 
Server: nginx/1.4.6 
Date: Mon, 25 Aug 2014 15:01:17 GMT 
Content-Type: text/html 
Content-Length: 154 
Connection: keep-alive 
NG: CCN-SW-1-5L2 
X-Mod-Name: GSLB/3.1.0 
Location: http://183.61.140.24/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 

HTTP/1.1 302 Moved Temporarily 
Server: nginx 
Date: Mon, 25 Aug 2014 15:01:17 GMT 
Content-Type: text/html 
Content-Length: 154 
Connection: keep-alive 
X-Mod-Name: Mvod-Server/4.3.3 
Location: http://183.61.140.20/a90a952900000000-1408984277-236096587/data1/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 
NG: CHN-ZJ-4-3M4 

HTTP/1.1 200 OK 
Server: nginx 
Date: Mon, 25 Aug 2014 15:01:17 GMT 
Content-Type: video/mp4 
Content-Length: 3706468 
Last-Modified: Mon, 25 Aug 2014 00:31:03 GMT 
Connection: keep-alive 
Cache-Control: no-cache 
ETag: "53fa83c7-388e64" 
NG: CHN-ZJ-4-3M4 
X-Mod-Name: Mvod-Server/4.3.3 
Accept-Ranges: bytes
```

可见，本次修改仍然没有成功！
为什么呢？分析一下，我们在@error\_page\_302这个location里已经加了一个头部打印语句，可是在测试中，该头部并没有打出来，可见流程并没有进入到@error\_page\_302这个location。

原因在于
```
error_page 302 = @error_page_302;
```
error\_page默认是本次处理的返回码。作为proxy，本次处理，只要转发上游服务器的响应成功，应该状态码都是200.即，我们真正需要检查的，是proxy的后端服务器返回的状态码，而不是proxy本身返回的状态码。查一下Nginx的wiki,proxy\_intercept\_errors指令正是干这个的:
```
Syntax:	proxy_intercept_errors on | off;
Default:	
proxy_intercept_errors off;
Context:	http, server, location
Determines whether proxied responses with codes greater than or equal to 300 should be passed to a client or be redirected to nginx for processing with the error_page directive.
```
## 3. 第二次修改
```
server {
        listen 80;
        proxy_intercept_errors on;
        location / {
                rewrite_by_lua '
                        ngx.exec("/proxy-to" .. ngx.var.request_uri)
                ';
        }
        location ~ /proxy-to/([^/]+)(.*) {
                proxy_pass http://$1$2$is_args$query_string;
                error_page 302 = @error_page_302;

        }
        location @error_page_302 {
                rewrite_by_lua '
                        local _, _, upstream_http_location = string.find(ngx.var.upstream_http_location, "^http:/(.*)$")
                        ngx.header["zzzz"] = "/proxy-to" .. upstream_http_location
                        ngx.exec("/proxy-to" .. upstream_http_location);
                ';
        }
}
```

与上一次修改相比，区别仅仅在于增加了一个proxy\_intercept\_errors指令。测试结果如下：
```
curl -I "http://192.168.109.128/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4" -L 
HTTP/1.1 302 Moved Temporarily
Server: nginx/1.4.6
Date: Mon, 25 Aug 2014 15:05:54 GMT
Content-Type: text/html
Content-Length: 160
Connection: keep-alive
zzzz: /proxy-to/183.61.140.24/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 
```
这次更神奇了，直接返回一个302状态完事，也不继续跳转了。
问题出在，虽然第一次302，请求成功的进入到@error\_page\_302，但后续的error\_page指令却没起作用。也就是说，error_page只检查了第一次后端返回的状态码，而没有继续检查后续的后端状态码。

查一下资料，这个时候，另一个指令 recursive\_error\_pages就派上用场了。

## 4. 第3次修改
```
server {
        listen 80;
        proxy_intercept_errors on;
        recursive_error_pages on;
        location / {
                rewrite_by_lua '
                        ngx.exec("/proxy-to" .. ngx.var.request_uri)
                ';
        }
        location ~ /proxy-to/([^/]+)(.*) {
                proxy_pass http://$1$2$is_args$query_string;
                error_page 302 = @error_page_302;

        }
        location @error_page_302 {
                rewrite_by_lua '
                        local _, _, upstream_http_location = string.find(ngx.var.upstream_http_location, "^http:/(.*)$")
                        ngx.header["zzzz"] = "/proxy-to" .. upstream_http_location
                        ngx.exec("/proxy-to" .. upstream_http_location);
                ';
        }
}
```

与上一次相比，仅仅增加了recursive\_error\_pages on这条指令。测试结果如下：
```
curl -I "http://192.168.109.128/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4" -L 
HTTP/1.1 200 OK 
Server: nginx/1.4.6 
Date: Mon, 25 Aug 2014 15:09:04 GMT 
Content-Type: video/mp4 
Content-Length: 3706468 
Connection: keep-alive 
zzzz: /proxy-to/14.18.140.83/f48bad0100000000-1408984745-236096587/data6/flv.bn.netease.com/tvmrepo/2014/8/5/P/EA3I1J05P/SD/EA3I1J05P-mobile.mp4 
Last-Modified: Mon, 25 Aug 2014 00:21:07 GMT 
Cache-Control: no-cache 
ETag: "53fa8173-388e64" 
NG: CHN-MM-4-3FE 
X-Mod-Name: Mvod-Server/4.3.3 
Accept-Ranges: bytes
```
可见，Nginx终于成功的返回200了。此时，Nginx才真正起到了一个Proxy的功能，隐藏了一个请求原本的多个302链路，只返回客户端一个最终结果。

## 5. 小结

综上，通过proxy\_pass、error\_page、proxy\_intercept\_errors、recursive\_error\_pages这几个指令的配合使用，可以向客户端隐藏一条请求的跳转细节，直接返回用户一个状态码为200的最终结果。

奇怪的是，在Nginx的官方wiki中并没有recursive\_error\_pages指令的相关说明。

