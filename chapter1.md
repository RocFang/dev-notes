# 网络编程

## 1. socket中的close与shutdown

shutdown的原型为：
```
#include <sys/socket.h>
int shutdown(int s, int how);
```
how可以为如下值：
1. SHUT_RD:调用后不能从该socket中读入数据。
2. SHUT_WR:调用后不能向该socket中写入数据。
3. SHUT_RDWR:综合1和2.即，既不能从该socket中读入数据，也不能向该socket中写入数据。

既然我们已经有了close,为什么还需要shutdown呢？主要有两点：
1. close本身并不会释放该socket占用的内存,而只是将该socket的引用计数减一。只有当该socket的引用计数减到0时,才会释放相关内存。
 所以，调用close并不能保证该socket不可用了，而调用shutdown，可以直接关闭该连接。

2.shudown可以单方向的关闭一个socket连接，由参数how控制。


