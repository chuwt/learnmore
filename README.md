# learnmore
dive into program
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
