[TOC]

# redis 实现原理

## 数据类型

### 字符串

- 采用简单动态字符串（SDS）

  - 数据结构

    ```
    len buf指向的字符数组的大小（不计算buf的最后的\0空字符)
    free buf指向的字符数组剩余的空间
    buf 指向字符数组的指针
    // 与golang的数组类似
    ```

  - 不实用c原生string的原因

    - c不保存字符串长度，获取需要遍历
    - 防止字符串溢出，通过判断free查看是否有足够的空间
    - 不通过\0判断buf截止，所以更通用
    - 优化内存分配，当字符串变动时进行内存分配
      - 空间预分配 用于增加操作
        - 修改后的buf > 1M, 预分配1M的free
        - 修改后的buf < 1M，预分配与len相同大小的的free
      - 惰性分配 用于缩短操作
        - 不重新分配buf大小，而是buf不变，修改free的大小

    

### 链表

- 实现
  - 数据结构（双向链表）

    - node

      ```
      prev 指针，指向上一个数据结构
      next 指针，指向下一个数据结构
      value 指针，值，可以指向不同类型的值
      ```

    - list

      ```
      head *node 头部指针
      tail *node 尾部指针
      len				 长度
      ```

### 字典

- 实现

  - hashtable

    ```
    table *dictEntry // hash表数组， dictEntry类型
    size	// hash表大小
    sizemask // 大小掩码，用于计算索引值，总是等于size-1，用于决定key应该放在哪个table索引上
    used	// hash表已经有的节点数量
    ```

  - dictEntry

    ```
    key
    value
    next	指向下一个节点，形成链表
    ```

  - 字典

    ```
    type	// 类型特定函数，操作函数
    privdata	// 私有数据
    ht[2]	// hash表，一般指使用ht[0]， ht[1]只会对ht[0]进行rehash时使用
    rehashidx // rehash索引，当rehash不在进行时，值为-1
    ```

- hash算法

  - 当一个新的key要添加到字典里面时，需要先根据key计算出hash和索引，根据索引插入到ht的table的指定索引上
  - hash函数根据type的不同，调用不同的hashFunction(key)
  - 索引Index = hash & dict->ht[x].sizemask, x 可能是0也可能是1
  - 当字典被作为数据库底层时，hash算法采用Murmuhash2
  - 负载因子：used/size

- 解决key冲突

  - 通过dictEntry的链表解决冲突（不同的key经过hash后得到相同的index）
  - 总是将新节点放在头部

- rehash

  - 为ht[1]分配空间
    - 扩张：第一个大于等于ht[0].used*2的2**N
    - 收缩：第一个大于等于ht[0].used的2**N
  - 将ht[0]的所有key，value重新hash到ht[1]上
  - 释放ht[0], 将ht[1]替换为ht[0], 重新生成ht[1]

- 什么时候执行rehash

  - hash表的负载因子大于1，同时服务器没有在bgsave或bgrewriteaof
  - 服务器在执行bgsave或bgrewriteaof，并且负载因子>=5
    - 避免在执行子进程时不必要的内存操作

- 渐进式rehash

  - 当ht[0]很大时，一次rehash到ht[1]会很长，所以采用渐进式rehash
  - 步骤
    - 为ht[1]分配空间
    - 将字典的rehashidx设置为0，表示rehash开始工作
    - 每次对字典的操作，都会额外执行rehash[rehashidx],将对应的索引的key和value rehash到ht[1]中

### 跳跃表（skiplist）

- 平均查找时间 O(logN), 最坏 O(N)

- 大部分情况与平衡术相媲美，不少程序使用跳跃表代替平衡树（b树）

- sorted set的底层实现

- 实现

  - 数据结构

    ```
    header // 指向表头
    tail		// 指向表尾
    level	// 跳跃表最大高度
    length	// 节点数量
    ```

  - 随机选取提拔的节点

### 整数集合

- 集合的底层实现之一（set）

- 实现

  - 数据结构

    ```
    encoding // 编码方式 int16 int32 int64
    length // 集合数量
    contents [] // 保存元素的数组，类型取决于encoding的类型,从小到大排序
    ```

  - 升级/降级

    - 修改encoding的编码方式以便节约内存和统一类型

### 压缩列表（ziplist）

- 为了节约内存

### 对象

- 引用计数的内存回收

- 引用技术的对象共享

- 数据结构

  ```
  redisObject{
  type 类型 // string, list, map, set, zset
  encoding 编码
  *ptr 指向底层实现数据结构的指针
  refcount 引用计数
  lru 最后一次被命令访问的时间, 用于缓存淘汰
  }
  ```

- 类型

  - string
  - list
  - map
  - set
  - zset

- 每种类型都至少使用了两种不同的编码
  - string
    - 整数值实现的字符串
    - embstr的简单动态字符串
    - 简单动态字符串
  - lit
    - 压缩列表实现的
    - 双端列表实现的
  - hash
    - 压缩列表实现
    - 字典实现
  - set
    - 整数集合实现
    - 字典实现
  - zset
    - 压缩列表实现
    - 跳跃表和字典实现

- 内存回收
  - `refcount` 引用计数
  - 创建时设置为1
  - 被使用+1
  - 不再使用-1
- 对象共享（类似python）
  - 指向引用计数 !=0的对象

## 单机数据库的实现

### 数据库

- 默认16个db
- 键值对数据库
- 过期键删除策略
  - 定时删除
    - 通过创建定时器
  - 惰性删除
    - 获取时判断是否过期
  - 定期删除
    - 每隔一段时间，统一删除
- redis使用惰性和定期删除

- 载入数据库时过期的键的影响
  - 执行save命令或者bgsave命令所产生的新RDB文件不会包含已过期的key
  - 执行bgrewriteaof所产生的重写AOF文件不会包含已过期的key
  - 当一个过期key删除之后，服务器会追加一条del命令到所有的AOF的末尾显式删除过期键
  - 主服务器删除一个过期键后，会发送del到所有从服务器

### RDB持久化

- 创建
  - SAVE 
    - 阻塞当前进程知道创建成功
  - BGSAVE
    - 创建子进程
- 保存
  - 设置保存条件，一般是多长时间内进行了多少次修改

### AOF持久化（append only file）

- 通过保存命令来进行持久化
- 每次时间循环都会调用aof函数
- 先将命令写入缓冲区，再通过aof函数同步到文件中

## 多机数据库的实现


