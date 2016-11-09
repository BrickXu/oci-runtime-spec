# Linux运行时

## 文件描述符

默认情况下，运行时为应用程序打开`stdin`，`stdout`和`stderr`的文件描述符。
运行时 **可能** 给应用程序传入额外的文件描述，以支持如[socket activation](http://0pointer.de/blog/projects/socket-activated-containers.html)等功能。
有些文件描述符 **可能** 被重定向到了 `/dev/null`，即使他们是打开的。

## 设备符号链接

在容器的`/proc`挂载后，**必须** 在`/dev`下为IO设置下列的标准符号链接。

|    Source       | Destination |
| --------------- | ----------- |
| /proc/self/fd   | /dev/fd     |
| /proc/self/fd/0 | /dev/stdin  |
| /proc/self/fd/1 | /dev/stdout |
| /proc/self/fd/2 | /dev/stderr |
