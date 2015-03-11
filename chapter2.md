# 计算机体系结构

## 1.大小端判断
定义：大端是高位在前，小端是低位在前。

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