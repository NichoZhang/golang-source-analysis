# sync-waitgroup
## 根据使用方式查看实现功能
使用实例(runtime/race/testdata/waitgroup_test.go)：
```
func TestNoRaceWaitGroup(t *testing.T) {
	var x int
	_ = x
	var wg sync.WaitGroup
	n := 1
	for i := 0; i < n; i++ {
		wg.Add(1)
		j := i
		go func() {
			x = j
			wg.Done()
		}()
	}
	wg.Wait()
}
```

## sync.WaitGroup 结构体
```
type WaitGroup struct {
    // Go语言特性，第一次使用后不可再复制，只能使用指针传递保证全局唯一
    noCopy noCopy 
    
    // 64位值：高32位是计数器，低32位用于计算等待的goroutine。
    // 64位原子操作需要进行64位对齐，
    // 但是，不支持32位系统。
    // 所以，我们分配了12bytes，使用其中的 8bytes 当作状态对齐位使用。
    // 其目的是在 12bytes 中至少含有一个 8bytes 用于对齐，具体：
    // 当指针位置指在（2n+1）的位置，抛弃前 4bytes ，使用后 8bytes；
    // 当指针位置指在（2n）的位置，使用前 8bytes 作为状态计数。
    // 00000000 00000000 …… 00000000 00000000
    // \_________ 8 bytes = 64 个 0 ________/
    state1 [12]byte

    sema uint32
}
```
从定义的结构体中可以看出，sync.WaitGroup 是全局唯一的，因为使用了noCopy，同时，在 计算等待goroutine 的位数可以同时等待 2^32 个goroutine，这也是可以使用waitGroup的最大数量。

## sync.WaitGroup.Add(delta int)
该函数是在循环中调用的，具体函数为：
```
// Add 添加delda到 WaitGroup 计数器，可以为负数。
// 如果计数器变为 零， 所有被阻塞的goroutine同时被释放。
// 如果计数结果位 负数，添加 panic。
// 注意，
// 
func (wg *WaitGroup) Add(delta int) {
    ……
    // 原子操作 加 1
    // 此时如果状态位 statep 为空，且 delta 等于 1，则操作原子加1结果为：
    // 00000000 00000000 00000000 00000001 00000000 …… 0000
    // \___________ 前32位 _______________/\__ 后32位均为0 _/
    // 当前状态位存在值 1，则再添加 delta 等于 1， 其结果为：
    // 00000000 00000000 00000000 00000010 00000000 …… 0000
    // \___________ 前32位 _______________/\__ 后32位均为0 _/
    state := atomic.AddUint64(satep, uint64(delta)<<32)
    ……
}
```

## sync.WaitGroup.Done()
```
// Add函数是加，Done函数是减，且是固定减 1 个。
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

## sync.Wait()
```
func (wg *WaitGroup) Wait() {
    statep := wg.state()
    ……
    for {
    ……
    // cas compare and swap 原子交换
    if atomic.CompareAndSwapUint64(statep, state, state+1) {
    ……
    }
    return 
    }
    ……
}
```
