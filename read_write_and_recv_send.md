# read/write与recv/send的区别

在功能上，read/write是recv/send的子集。read/wirte是更通用的文件描述符操作，而recv/send在socket领域则更“专业”一些。

当recv/send的flag参数设置为0时，则和read/write是一样的。

如果有如下几种需求，则read/write无法满足，必须使用recv/send:
1. 为接收和发送进行一些选项设置
2. 从多个客户端中接收报文
3. 发送带外数据（out-of-band data)


