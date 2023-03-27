### ulimit

``ulimit -a`是一个Linux命令，用于显示当前用户进程的所有限制。这个命令会显示当前用户进程的所有限制，包括进程可用的CPU时间、最大可用的内存、可用的文件描述符数目、可以打开的文件大小限制等等。这些限制是为了保证系统的稳定性和安全性而设定的，可以防止某个进程使用过多的系统资源而影响其他进程的运行。

输出结果通常是一个列表，它包含当前用户进程的所有限制，例如：

```c
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
file size               (blocks, -f) unlimited
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
...
```

在这个例子中，`-n`参数表示当前用户进程所能打开的最大文件描述符数目是1024。其他的参数说明了关于内存、文件大小、CPU时间等方面的限制。需要注意的是，这些限制通常在/etc/security/limits.conf文件中设定，可以通过修改这个文件来自定义用户进程的限制。



### top

`top -H -p pid`：查看对应进程的线程页面

