# listen中的backlog参数的含义

listen的原型为:
```
#include <sys/socket.h>

int listen(int sockfd, int backlog);
```
其中backlog有什么意义呢？