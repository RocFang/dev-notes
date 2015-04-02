# 关于proxy_pass的参数路径问题

由于工作需要，开始分析nginx的proxy模块，在分析之前，当然要先会用了。于是开始熟悉该模块的一些指令，其中最基本的指令要属proxy\_pass了。nginx的英文文档总是看着感觉有些别扭，于是按惯例先google了一些文章。

这一搜，就掉进坑里了。

这些文章里都把proxy\_pass的目标地址是形如“127.0.0.1:8090”和“127.0.0.1:8090/”分开讨论，认为后者“/"的作用是删除url中匹配的部分，然后再讨论目标地址中带了uri的情况。

其实根本没这么复杂，只有两种情况：

（1）目标地址中不带uri。即proxy\_pass的参数形如"http://127.0.0.1:8090"。
此时新的目标url中，匹配的uri部分不做修改，原来是什么样就是什么样。

（2）目标地址中带uri。即proxy\_pass的参数形如“http://127.0.0.1:8090/dir1/dir2"

此时新的目标url中，匹配的uri部分将会被修改为该参数中的uri，如"http://127.0.0.1:8888/dir1/dir2."

有人说，你没有讨论ip和端口后带不带”/“的区别。其实是不需要的，因为"/"本身就是一种uri，很明显属于上面的第二种情况，只不过是把原来的uri修改为了现在的uri（”/")，看上去，像是删除了原url中匹配的部分。如果不理解这一点，就会总想着去牢记、区分结尾带不带"/"的情况。
官方文档也是这么叙述的，根本没有提及半句“/"：

>A request URI is passed to the server as follows:
>
>If the proxy\_pass directive is specified with a URI, then when a request is passed to the server, the part of a normalized request URI matching the location is replaced by a URI specified in the directive:
location /name/ {
    proxy\_pass http://127.0.0.1/remote/;
}
If proxy\_pass is specified without a URI, the request URI is passed to the server in the same form as sent by a client when the original request is processed, or the full normalized request URI is passed when processing the changed URI:
location /some/path/ {
    proxy\_pass http://127.0.0.1;
}

测试部分如下。

如果配置为：
```
server {
                listen 9090;
                access\_log /home/strider/project/nginx/nginx-1.4.2/log/access\_9090.log;
                location /test1/test2/{
                    proxy\_pass http://127.0.0.1:8090;
                }   
        }   
```
则有如下对应关系：
```
127.0.0.1:9090/test1/test2/echo1----->127.0.0.1:8090/test1/test2/echo1
127.0.0.1:9090/test1/test2/---->127.0.0.1:8090/test1/test2
```
如果配置为：
```
server {
                listen 9090;
                access\_log /home/strider/project/nginx/nginx-1.4.2/log/access\_9090.log;
                location /test1/test2/{
                    proxy\_pass http://127.0.0.1:8090/;
                }   
}   
```
则有如下对应关系：
```
127.0.0.1:9090/test1/test2/echo1----->127.0.0.1:8090/echo1
127.0.0.1:9090/test1/test2/---->127.0.0.1:8090/
```

如果配置为：
```
server {
                listen 9090;
                access\_log /home/strider/project/nginx/nginx-1.4.2/log/access\_9090.log;
                location /test1/test2/{
                        proxy\_pass http://127.0.0.1:8090/test1;
                }   
}  
```
则有如下对应关系：
```
127.0.0.1:9090/test1/test2/echo1----->127.0.0.1:8090/test1echo1
127.0.0.1:9090/test1/test2/---->127.0.0.1:8090/test1
```
如果配置为：
```
server {
                listen 9090;
                access\_log /home/strider/project/nginx/nginx-1.4.2/log/access\_9090.log;
                location /test1/test2/{
                        proxy\_pass http://127.0.0.1:8090/test3/test4/test5;
                }   
}   
```
则有如下对应关系：
```
127.0.0.1:9090/test1/test2/echo1----->127.0.0.1:8090/test3/test4/test5echo1
127.0.0.1:9090/test1/test2/---->127.0.0.1:80990/test3/test4/test5
```