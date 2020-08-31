# evio
最近读了evio的源码，学习了一下创建异步读写的tcp server
# 原因
golang 自带的net/http中对于读写的操作是非阻塞的，但是我们实际使用的时候是阻塞的，
原因是当读写时fd返回状态为EAGAIN时，调用gopark挂起当前goroutine，达到同步的目的，当连接很多时，会浪费很多goroutine的调度开销，所以有了使用异步读写，使用一个线程进行读写和连接
# 架构图
![evio架构图](https://github.com/chuwt/learnmore/tree/master/imgs/evio.jpg)
