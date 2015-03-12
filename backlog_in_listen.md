# listen中的backlog参数的含义

listen的原型为:
```
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```
其中backlog有什么意义呢？

关于这个参数的很多说法都很模糊。

在《unix环境高级编程》第三版中是这么说的:
>The backlog argument provides a hint to the system regarding the number of outstanding connect requests that it should enqueue on behalf of the process. The actual value is determined by the system, but the upper limit is specified as SOMAXCONN
in <sys/socket.h>.

连"hint"这种词都用上了，说的是相当模糊，对吗？

我们再来看看《unix网络编程》第三版:

>The listen function is called only by a TCP server and it performs two actions:

>1 . When a socket is created by the socket function, it is assumed to be an active
socket, that is, a client socket that will issue a connect. The listen function
converts an unconnected socket into a passive socket, indicating that the kernel
should accept incoming connection requests directed to this socket. In terms of the TCP state transition diagram (Figure 2.4), the call to listen moves the socket from
the CLOSED state to the LISTEN state.

>2 . The second argument to this function specifies the maximum number of connections
the kernel should queue for this socket.

>This function is normally called after both the socket and bind functions and must be
called before calling the accept function.


