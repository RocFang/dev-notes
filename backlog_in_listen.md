# listen中的backlog参数的含义

listen的原型为:
```
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```
其中backlog有什么意义呢？

关于这个参数的很多说法都很模糊。

在《unix环境高级编程》第三版中是这么说的:
>The backlog argument provides a hint to the system regarding the number of
outstanding connect requests that it should enqueue on behalf of the process. The
actual value is determined by the system, but the upper limit is specified as SOMAXCONN
in <sys/socket.h>.