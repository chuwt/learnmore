# 内存对齐
golang会自动对齐内存，但是结构体字段的顺序不同会导致结构体内存大小不同
```
type TestA struct {
    A int32 // 4
	B int64 // 8
    C int32 // 4
}
// 占24位，因为A和C不足8位则补齐8位
unsafe.Sizeof(TestA{})

type TestB struct {
    A int32 // 4
    C int32 // 4
	B int64 // 8
}

// 占16位，因为A和C正好占8位，不需要补齐
unsafe.Sizeof(TestB{})
```

# 缓存对齐
cache line一般为64位
```
type TestA struct {
    A int32 // 4
    _ [60]byte // 后移60位
    B int64 // 8
    _ [56]byte // 后移56位
}
此时对A和B的并发操作的性能优于没有对齐的操作
```
