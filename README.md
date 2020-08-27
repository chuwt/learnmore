# learnmore
dive into program
# 基础数据结构
## 数组
- 数据结构
```
elem  元素
bound 大小
```
- 内存分配
  - 长度<=4时放入栈中，长度>4时放入堆中
- 内存固定大小无法修改
## 切片
- 数据结构
```
data  数据连续内存的指针
len   大小
cap   容量
```
- 内存分配
  - 没有发生逃逸且很小时放入栈
  - 发生逃逸或者特别大时放入堆
- 追加元素
  - len<=cap时直接在data内存后加入
  - len>cap时重新分配连续内存，然后data数据写入新内存
    - 当期望容量大于当前容量的两倍，则使用期望容量
    - 当前切片小于1024，则翻倍
    - 当前切变大于1024，则每次新增25%,知道cap>len
- copy
  - cap和len都到最小的为止
## map
- 数据结构
```
mutex
hash种子
桶
溢出桶
```
- 原理
  - 通过hash函数进行归类查找
- 实现方式
  - 开放寻址
    - 一个数据，重复的往后延
  - 拉链算法（链表）
    - 通过hash后放入指定链表中，一般链表长度不会太长
    - 有些算法做了红黑树优化
## string
```
data
len
```
# 函数
- 传递方式
  - 值传递
- 闭包
  - 闭包内引用的外部变量全部是指针（引用）
  - 所有的匿名函数都是闭包函数
- defer
  - 无参数传递的defer，在defer定义的地方就会将参数的值进行缓存，defer执行时使用的是缓存的值
  - 匿名的defer函数（闭包），则使用运行时的值
# interface
- 分类
  - 无方法的eface
    ```
    _type 类型指针
    data  底层数据指针
    ```
  - 带方法的iface
    ```
    tab   类型方法等
    data  底层数据指针
    ```
- iface中结构体方法和指针方法的区别
  - 结构体方法接收的是结构体，所以传入指针报错
  - 指针方法接受的是指针，但是如果调用的是结构体，则会隐式的进行调用
- 反射
  - reflect.TypeOf
  - reflect.ValueOf
  - 可以看一下gin的bind实现
# runtime
## time定时器
  - 版本
    - 1.9之前
      - 采用全局四叉堆，需要加锁
    - 1.13之前
      - 全局64个四叉堆，p根据hash调用对应的四叉堆，线程切换频繁
    - 1.14 
      - 每个p都有一个四叉堆，通过网络轮询器触发
  - 数据结构
    - 最小四叉堆
  - 全局四叉堆原理
    - 当时间触发时，创建timeProc进行阻塞当前M，切出P，保存G，其他M接管执行当前P。
    当时间再次触发，恢复M，M获取P，G插入到P最前端
  - 网络轮询原理
    - P调度G时都会查询当前堆中是否存在到期的定时器，存在则进行调度，同时会尝试偷取其他P的堆，防止当前P
    调用的G无法让出资源导致定时器过期
  
## go调度
## go gc
## go内存
## context
- 一般用于并发控制
- WithCancel
- BackGroud/TODO
- WithValue
  - 会向父级查找
- WithTimeout/WithDeadline
## sync
- 锁
  - 互斥锁
    - Mutex
        - 上锁使用CAS实现
        - 饥饿模式防止goroutine饿死
        - 自旋获取锁
  - 读写锁
    - RWMutex
        - 读锁采用AddInt32实现
- pool
  - 对象池
  - 用法
    ```
    sp := sync.Pool{
      New: func() {
        reutrn make([]byte, 1024)
      }
    }
    sp.Get()
    sp.Put(make([]byte, 1024))
    ```
  - 实现原理
    - 1.13
      - 每个p都会存在一个private和shared的缓存池，当前p获取时先从自己的private拿，如果没有，去shared取，如果shared没有，则去其他p的shared去取
      - priavte不需要锁，shared需要锁
      - 所以如果shared为空，则需要遍历所有p的shared，导致性能下降
    - 1.14
      - private和shared合并为local
      - 通过双向链表的方式获取缓存对象，当前p从链表头获取，其他p从链表尾部获取
- WaitGroup
  - 并发控制
- singleflight.Group
  - 用于相同的请求（调用）限制
  - 原理
    - 设置请求的key，其他请求进来后，如果存在当前key，则等待第一个key的调用结束，获取返回
## channel
# 源码
## net/http

# 内存重排
- 多线程时，cpu将耗时的数据进行缓存（store缓存），继续执行其他命令，其他命令进行读取的时候，发现memoy中没有，因为在store buffer中。解决方法时加入锁
- 只会对后面没有用到的数据进行重拍，如果用到了，则不会等待数据从store buffer 传播到内存中
- 例子
  ```
  var x, y int
	go func() {
		x = 1 // A1
		fmt.Print("y:", y, " ") // A2
	}()
	go func() {
		y = 1                   // B1
		fmt.Print("x:", x, " ") // B2
	}()
  会存在
  x:0 y:0的结果
  ```
# 缓存对齐，内存对齐
- 内存对齐的原理	
  - 内存又8个chip组成，每个chip下又有8个bank，所以一次能并行读取64byte
  - 如果不对齐，则一个8byte的数据可能需要读2次
  - 如果对齐，则只需要读1次
- 缓存对齐
  - cpu的缓存都有cache line，一般64位为64byte，所以当读取某一数据时，会将相邻的数据一并读入cache line
  - 如果缓存没有对齐，则当并行修改时，Acpu修改了a，如果Bcpu也读入了a，则需要将Bcpu的a进行缓存一致性同步，导致性能下降（false sharing）
    - 真共享
      - 不同的cpu共享数据，存在数据同步
    - 假共享（false sharing）
      - 不同的cpu不共享数据，但是读入cache line时读入了其他cpu的数据，则需要进行数据同步，可以通过缓存对齐优化，取消数据同步
    - 缓存一致性协议MESI和MOESI
      - MESI
      - MOESI
  - 缓存淘汰策略
    - lru
  - 缓存写入主存策略
    - 直接写
    - 存入写队列等待写
    - 标记dirty等待写
