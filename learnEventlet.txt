## Python Eventlet 

by leannmak 2015-7-24

### 协程

三种并发开发模型：进程、线程、异步IO

进程、线程与协程的包含关系：进程（线程（协程）））。

但事实上，协程主要用于替代多线程处理IO wait，以达到无阻塞IO的效果，因此并 **不推荐协程与线程同用**。

* 协程不是专用的，只适合于特定情况，一般用于处理操作系统和网络IO。
* 协程对函数的执行进行切片，不占时间，但吃内存。
* CPU占用率高的逻辑运算（如人工智能算法等）请使用多线程处理。

协程最主要的两大优点在于：`高效处理IO` 和 `高效编程`。

协程是应用态的函数，有自己的堆栈和寄存器，一个进程里可以包含多个协程，\
但同一时间一定只有一个协程在执行，因此协程无需理会全局变量（不用加锁）。

在linux内核态中，每一个线程都被视为一个进程在执行，而协程在内核态中是不可见的，\
无法利用内核调度机制，因此协程的切换是不确定的，协程之间不存在优先级。

OpenStack 并发采用 `多进程` + `多协程` 模式。


### Greenlet

通过 `switch()` 控制协程的切换，较底层，一般不用：
```bash
from greenlet import greenlet

def test1():
    print 12
    gr2.switch()
    print 34

def test2():
    print 56
    gr1.switch()
    print 78

gr1 = greenlet(test1)
gr2 = greenlet(test2)
gr1.switch()
```
运行结果：
```
12
56
34
```
Github: https://github.com/python-greenlet/greenlet


### Eventlet

在eventlet里，把 “协程” 叫做 `greenthread`(绿色线程)。

对python的 **标准库**（os）进行了封装（第三方库不一定支持），采用打补丁的方式，\
导入后无需改动原有代码，直接变成非阻塞IO:
```bash
import eventlet
eventlet.monkey_patch(all=True)
```
也可采用 `eventlet.sleep(0.01)` 执行一次协程切换，`0.01`表示0.01s后可以切回来，\
相当于 `greenlet.switch()`。

参考链接：http://eventlet.net/
