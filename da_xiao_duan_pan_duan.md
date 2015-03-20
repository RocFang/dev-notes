# 大小端判断
定义：大端是高位（MSB）在地址靠前的部分，小端是低位(LSB)在地址靠前的部分。网络字节序是大端序，x86是小端序。

判断方法：
```
#include<stdio.h>

int main()
{
    int i = 0x11223344;
    char *p;

    p = (char *) &i;
    if (*p == 0x44)
    {
        printf("%s\n", "Little Endian!");
    }
    else
    {
        printf("%s\n", "Big Endian!");
    }
}
```

