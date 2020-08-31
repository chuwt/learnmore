# evio
最近读了evio的源码，学习了一下创建异步读写的tcp server
# 原因
golang 自带的net/http中对于读写的操作是非阻塞的，但是我们实际使用的时候是阻塞的，
原因是当读写时fd返回状态为EAGAIN时，调用gopark挂起当前goroutine，达到同步的目的，当连接很多时，会浪费很多goroutine的调度开销，所以有了使用异步读写，使用一个线程进行读写和连接
# 架构图
![image](https://s1.ax1x.com/2020/08/31/djiOVU.jpg)
# 部分说明
- Kqueue/epoll
    - 主要对象
- Kevent
    - 监听事件
    - EADD/EDELETE // 添加删除事件
    - EVFILT_WRITE // 读写监听
    - changes // 监听的事件列表，内核以红黑树存储
    - events // 触发的events列表，内核以链表存储
