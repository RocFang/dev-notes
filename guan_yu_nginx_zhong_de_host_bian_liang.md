# 关于nginx中的host变量
关于变量host，在Nginx的官网wiki中是如下说明的：

>$host：in this order of precedence: host name from the request line, or host name from the “Host” request header field, or the server name matching a request

直白的翻译一下:host变量的值按照如下优先级获得：
1. 请求行中的host.
2. 请求头中的Host头部.
3. 与一条请求匹配的server name.

很清楚，有三点，取优先级最高的那个。仅从字面意思上来理解，这个选择的过程为：如果请求行中有host信息，则以请求行中的host作为host变量的值（host与host变量不是一个东西，很拗口）；如果请求行中没有host信息，则以请求头中的Host头的值作为host变量的值；如果前面两者都没有，那么host变量就是与该请求匹配所匹配的serve名。

为了表示语气的加强，再重复一遍，这个规则包含了**三个**步骤。

但由于某种不知名的原因，网上能找到的大部分关于该变量的说明都只包含两点。

例如这个：
>$host
>
>请求中的Host字段，如果请求中的Host字段不可用，则为服务器处理请求的服务器名称。

再例如这个：
>$host, 请求信息中的"Host"，如果请求中没有Host行，则等于设置的服务器名;

**这些说法都是错误的**。它们将一个本来很有内涵的东西向读者隐藏起来，如果你只是一扫而过，在心里默念，哦，这个变量原来是这样的，很简单嘛，那就会错过很多东西。下面就来详细的说明一下该变量。

## 1.什么是请求行中的host?
我们知道，HTTP是一个文本协议，建立在一个可靠的传输层协议之上。这个传输层协议要是可靠的，面向连接的。由于TCP的普及程度，让它成了HTTP下层协议事实上的标准。但我们要知道，HTTP并不仅限于建立在TCP之上。只要是可靠的，面向连接的传输层协议，都可以用来传输HTTP。下面所说的HTTP，都是指搭载在TCP之上的HTTP。

一个HTTP请求过程是这样的，客户端先与服务器建立起TCP连接，然后再与服务器端进行请求和回复的收发。请求包含请求行、请求头和请求体，其中，根据请求方法的不同，请求体是可选的。

下面开始说请求行。

在发送请求行之前，客户端与服务器已经建立了连接。所以此时请求行中并不需要有服务器的信息。例如，可以如下：
> GET /index.php HTTP/1.1

这就是一个完整的HTTP请求行。虽然请求行中不需要有服务器的信息，但仍然可以在请求行中包含服务器的信息。例如：
>GET www.pureage.info/index.php HTTP/1.1

两者一比较，就很容易理解什么叫请求行中的host了。第一个请求行中，就没有host，第二种请求行中，就带了host，为wwww.pureage.info。

## 2.Host请求头与HTTP/1.0、HTTP/1.1
一个请求，请求行下面就是一些列的请求头。这些请求头，在HTTP/1.0中，都是可选的，且HTTP/1.0不支持Host请求头；而**在HTTP/1.1中，Host请求头部必须存在**。

除了阅读RFC2616来确认这一点外，我们来看看Nginx中的代码：
```
ngx_int_t
ngx_http_process_request_header(ngx_http_request_t *r)
...(略)...
if (r->headers_in.host == NULL && r->http_version > NGX_HTTP_VERSION_10) {
        ngx_log_error(NGX_LOG_INFO, r->connection->log, 0,
                   "client sent HTTP/1.1 request without \"Host\" header");
        ngx_http_finalize_request(r, NGX_HTTP_BAD_REQUEST);
        return NGX_ERROR;
...(略)...
```
即，如果协议版本在HTTP/1.0之上，且请求头部没有Host的话，会直接返回一个Bad Request错误响应。

## 3.什么是与请求匹配的server name?
server name是指在Nginx配置文件中，在server块中，用server_name指令设置的值。一个server可以多次使用server_name指令，来实现俗称的“虚拟主机”。例如：
```
server {
    listen      80;
    server_name example.org www.example.org;
    ...
}

server {
    listen      80;
    server_name example.net www.example.net;
    ...
}

server {
    listen      80;
    server_name example.com www.example.com;
    ...
}
```
关于虚拟主机的确定方法，还是引用Nginx的官方文档：
>在这个配置中，nginx仅仅检查请求的“Host”头以决定该请求应由哪个虚拟主机来处理。如果Host头没有匹配任意一个虚拟主机，或者请求中根本没有包含Host头，那nginx会将请求分发到定义在此端口上的默认虚拟主机。在以上配置中，第一个被列出的虚拟主机即nginx的默认虚拟主机——这是nginx的默认行为。而且，可以显式地设置某个主机为默认虚拟主机，即在"listen"指令中设置"default_server"参数：
>
>server {
>    listen      80 default_server;
>    server_name example.net www.example.net;
>    ...
>}

## 4.整理一下

上面所有的说明，都仅仅是解释了Nginx官方文档中关于host变量选择的三个来源值。不过，知道了这三个值是什么，怎么来的，以及HTTP/1.0与HTTP/1.1的区别，我们就能彻底搞清楚这个host变量是怎么确定的了。

首先，无论是HTTP/1.0还是HTTP/1.1请求，只要请求行中带了主机信息，那么host变量就是该请求行中所带的host。需要注意的是，host变量是不带端口号的。

所以，下面两条请求的host变量都是www.pureage.info:
>GET http://www.pureage.info/index.php HTTP/1.0

>GET http://www.pureage.info/index.php HTTP/1.1

其次，假设请求行中不带主机信息，那么我们就来看请求头部中的Host头。此时HTTP/1.0请求和HTTP/1.1请求的表现就大不相同了。有如下几种情况：

1.HTTP/1.0请求，不带Host头
>GET /index.php HTTP/1.0

此时，host变量为与该请求匹配的虚拟主机的主机名，及在nginx配置文件中与之匹配的server段中server_name指令设置的值。如果该server中没有使用`server_name`指令，那么host变量就是一个空值，不存在。

2.HTTP/1.0请求，带Host头
>GET /index.php HTTP/1.0
>Host: www.pureage.info

此时，host变量即为www.pureage.info

3.HTTP/1.1请求，不带Host头
>GET /index.php HTTP/1.1

如前所述，HTTP/1.1请求必须携带Host头部。所以这种情况，请求会返回一个Bad Request错误响应。host变量当然就更不存在了。

4.HTTP/1.1请求，带Host头
>GET /index.php HTTP/1.1
>Host: www.pureage.info

此时，host变量即为www.pureage.info

通过上面的例子，可以看出，对于HTTP/1.1请求，host变量不会为空，它要么在请求行中指定，要么在Host头部指定，要么就是一条错误的请求；而对于HTTP/1.0请求，host变量有为空的情况，即请求行和请求头中没有指定host，且匹配的server也没有名称时。

## 5.验证
可以搭建一台Nginx服务器，用telnet来验证上面的说法。此处略过，但这个操作很有必要。


