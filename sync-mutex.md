# sync-mutex
根据使用方式逐步分析
``` go
func main() {
    var state = make(map[int]int)

    var mutex = &sync.Mutex{}

    var readOps uint64
    var writeOps uint64

    for r := 0; r < 100; r++ {
        go func() {
            total := 0
            for {
                key := rand.Intn(5)
                mutex.Lock()
                total += state[key]
                mutex.Unlock()
                atomic.AddUint64(&readOps, 1)

                time.Sleep(time.Millisecond)
            }
        }()
    }

    for w := 0; w < 10; w++ {
        go func() {
            for {
                key := rand.Intn(5)
                val := rand.Intn(100)
                mutex.Lock()
                state[key] = val
                mutex.Unlock()
                atomic.AddUint64(&writeOps, 1)
                time.Sleep(time.Millisecond)
            }
        }()
    }

    time.Sleep(time.Second)

    readOpsFinal := atomic.LoadUint64(&readOps)
    fmt.Println("readOps:", readOpsFinal)
    writeOpsFinal := atomic.LoadUint64(&writeOps)
    fmt.Println("writeOps:", writeOpsFinal)

    mutex.Lock()
    fmt.Println("state:", state)
    mutex.Unlock()
}

```

## sync.Mutex结构体
``` go
// mutex 是一个互斥锁。
// 当值为 零 时，代表尚未锁定。
// mutex 在第一次使用后就不可以被值拷贝。注：其他struct中nocopy的实现。
type Mutex struct {
    // 使用位来表示不同状态
    // 根据const赋值可以得到，从最低位到高位依次是：
    // 00000000 00000000 00000000 00000000
    // \______________29______________/|||
    //                                 ||是否锁定，0未锁定，1锁定
    //                                 |是否被唤起，0未唤起，1唤起
    //                                 是否饥饿，0未饥饿，1饥饿
    state int32
    sema  uint32
}

// 一个 Locker 代表一个对象是被锁定还是未被锁定。
type Locker interface {
    Lock()
    Unlock()
}
```
结构体中表示，目前最多可以接收2^29个goroutines的队列。

## CONST 
``` go
const (
    // mutextLocked 的位值为 0000 0001；
    // mutex.state & mutexLocked 得到加锁状态；
    // mutexLocked 1表示加锁，0表示未加锁。
    mutexLocked = 1 << iota

    // mutexWorken 的位值为 0000 0010；
    // mutex.state & mutexWorken 得到唤醒状态；
    // mutexWorken 1表示已唤起，0表示未唤起。
    mutexWorken

    // mutexStarving 的位值为 0000 0100；
    // mutex.state & mutexStarving 得到饥饿状态；
    // mutexStarving 1表示饥饿，0表示正常。
    mutexStarving

    // mutexWaiterShift 的值为 3；
    // mutex.state >> mutexWaiterShift 得到等待的goroutine数量。
    mutexWaiterShift = iota

    // 如果超过1e6（1毫秒），则状态转变为饥饿。
    starvationThresholdNs = 1e6
)
```


## sync.Mutex.Lock()
``` go
func (m *Mutex) Lock() {
    // 捷径：抢占未锁定的mutex。
    // 原子操作（CAS），当前值为 0 时，交换并保证状态值修改为 mutexLocked，返回锁定。
    // 此时，该 goroutine 的 state 值为 1。
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        return
    }

    // 如果判断失败，表示当前goroutine已经是锁定状态。
    var waitStartTime int64
    // 初始饥饿状态为 否
    starving := false
    // 初始激活状态 否
    awoke := false
    iter := 0
    // 将当前状态赋值给 old，当前状态为 1，所以 old 此时为 1。 
    old := m.state
    for {
        // 在进入饥饿模式后不要自旋，控制权交给等待的goroutines。
        // 所以，我们将不会获得互斥锁。
        // 已经锁定且非饥饿模式，并且可以使用自旋操作判断位真
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 使用自旋锁是一种常规，并且有效的方式。
            // 操作 mutexWoken 位，来通知 Unlock 操作，
            // 不要唤醒其他阻塞的goroutines。
            // 如下判断为“真”时需要：
            // 处于非激活状态 并且 当前goroutine还没被唤醒 并且 已经在排队 并且 可以被唤醒。
            // 将激活状态设置成位激活，或者说是唤醒。
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            // 开始自旋处理，调用汇编码进行加锁
            runtime_doSpin()
            // 循环加一，记录循环等待次数
            iter++
            old = m.state
            continue
        }
        new := old
        // 不要试着获取饥饿锁，新到达的goroutines一定会排到前面。
        if old&mutexStarving == 0 {
            new |= mutexLocked
        }
        // 如果处在饥饿状态或者是锁定状态，在等待位加一
        if old&(mutexLocked|mutexStarving) != 0 {
            new += 1 << mutexWaiterShift
        }
        // 当前的goroutine将锁的模式转变为饥饿模式。
        // 但是，如果当前goroutine还未锁定，就不会转换成饥饿模式。
        // 当在饥饿模式下有需要解锁的goroutines时，不会判断为真。
        // 饥饿模式，并且已经锁定，将new添加到饥饿状态队列
        if starving &&  old&mutexLocked != 0 {
            new += mutexStarving
        }
        if awoke {
            // groutine已经从休眠中唤醒，
            // 所以，我们需要重置状态位置
            if new&mutexWoken == 0 {
                trow("sync:inconsistent mutex state")
            }
            new &^ = mutexWoken
        }
        // 如果目前状态同old状态一致，并可以将new的状态赋值，为真
        if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 赋值之前的goroutine状态，如果是非锁定或者是非饥饿
            // 重新排队
            if old&(mutexLocked|mutexStarving) == 0 {
                break // 通过原子锁进行锁定
            }
            // 如果之前已经开始等待，插队到队伍的最前面，实现 LIFO
            queueLifo := waitStartTime != 0
            if waitStartTime == 0 {
                waitStartTime = runtime_nanotime()
            }
            runtime_SemacquireMutex(&m.sema, queueLifo)
            starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
            old = m.state
            if old&mutexStarving != 0 {
                // 如果goroutine被唤起，同事是饥饿模式的时候，
                // mutex的所有权会递回给我们，但是mutex的状态码错误了
                // 我们没有将当前goroutine进行锁定，
                // 当前goroutine还是作为一个排队队员。已经修复。
                if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                    throw("sync: inconsistent mutex state")
                }
                delta := int32(mutexLocked - 1<<mutexWaiterShift)
                if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式。
                    // 这里的判断很重要，并且要关注等待时间。
                    // 饥饿模式是很低效(所以要尽量快速退出饥饿模式)，
                    // 一旦有两个goroutines从mutex切换到饥饿模式，可能造成无限循环。
                    delta -= mutexStarving
                }
                atomic.AddInt32(&m.state, delta)
                break
            }
            awoke = true
            iter = 0
        } else {
            old = m.state
        }
    }
}
```

## mutex.Unlock()
``` go
func (m *Mutex) Unlock() {
    // 捷径：修改锁定位状态。
    new := atomic.AddInt32(&m.state, -mutexLocked)
    // 如果是未锁定或者是已经解锁的话，会提示错误。
    if (new+mutexLocked)&mutexLocked == 0 {
        throw("sync: unlock of unlocked mutex")
    }
    if new&mutexStarving == 0 {
        old := new
        for {
            // 如果没有等待的goroutines，或者当前goroutine已经唤醒
            // 或者已经抢到了锁，不在需要唤醒其他goroutines。
            // 在饥饿模式下，锁有钱直接递交到下一个未锁定的goroutine。
            // 当我们解锁mutex之前，我们没有观察到mutexStarving状态，那我们就不是这个链条上的一部分。
            // 所以退出。
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // 向右唯一，唤醒下一位。
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false)
                return
            }
            old = m.state
        }
    } else {
        // 饥饿状态：递交所有权到下个等待的goroutine。
        // 注意：mutexLocked 没有设置，下一个goroutine将会在唤醒后设置。
        // 但是如果mutexStarving被设置，mutex依然会被认为是锁定的，
        // 所以当新到的goroutines不会被接收。
        runtime_Semrelease(&m.sema, true)
    }
}
```

## 关于 const中的大量注释
> Mutex can be in 2 modes of operations: normal and starvation.
> 
> In normal mode waiters are queued in FIFO order, but a woken up waiter does not own the mutex and competes with new arriving goroutines over the ownership. New arriving goroutines have an advantage -- they are already running on CPU and there can be lots of them, so a woken up waiter has good chances of losing. In such case it is queued at front of the wait queue. If a waiter fails to acquire the mutex for more than 1ms, it switches mutex to the starvation mode.  
> 
> In starvation mode ownership of the mutex is directly handed off from the unlocking goroutine to the waiter at the front of the queue.  New arriving goroutines don't try to acquire the mutex even if it appears to be unlocked, and don't try to spin. Instead they queue themselves at the tail of the wait queue.
> 
> If a waiter receives ownership of the mutex and sees that either (1) it is the last waiter in the queue, or (2) it waited for less than 1 ms, it switches mutex back to normal operation mode.
>  
> Normal mode has considerably better performance as a goroutine can acquire a mutex several times in a row even if there are blocked waiters.  Starvation mode is important to prevent pathological cases of tail latency.
翻译如下
> Mutex有两种操作模式：正常 和 饥饿。
> 
> 在正常模式下，等待的goroutines按照 FIFO（先进先出）的顺序排队执行，但是，goroutine被唤醒后不能拥有mutex，需要同新到达的gotoutines进行竞争。然而，新到达的goroutines拥有一个优势，它已经在CPU上运行了，并且可能数量还不少，所以，当一个刚被唤醒的goroutine将会很可能竞争失败。在这种情况下，刚被唤醒的goroutine就会排在队列的最前面。如果一个goroutines超过1ms没有获得mutex，将会从正常模式，转换成饥饿模式。
> 
> 在饥饿模式下，mutex的所有权会直接从刚解锁的goroutine中递交到队列最前面的gotoutine手上。新到达的gotoutines不会尝试去获取mutex，即使它没有锁定，也会去尝试自旋。相反，他们会排队到等待队列的尾部。
> 
> 如果一个goroutine拥有了mutex的所有权，并有如下两种情况之一，就会从饥饿模式变化普通模式：
> 1）它是队列中的最后一个；
> 2）它的等待时间小于1ms。
> 
> 正常模式拥有更好的性能，一个goroutine可以获取更多次的mutex，即使它阻塞了其他goroutines。
> 饥饿模式对于预防病态的尾部延迟也很重要（由于饥饿模式会跳过超时goroutine，让队列一直处于运行状态）。
> 
## 源代码中的建议
> Package sync provides basic synchronization primitives such as mutual exclusion locks. Other than the Once and WaitGroup types, more are intended for use by low-level library routines. Higher-level synchronization is better done via channels and communication.
翻译如下：
> sync包中提供基础的同步操作，例如互斥锁。还有更多方便底层库使用的方法，例如 Once 和 WaitGroup 的类型等等。高层同步方式最好通过 channels 和 communication 来实现。

Channel 是 Golang 中使用的一种特殊的消息通信的方式，类似于 管道队列；
至于其他的 Communication 还有 共享内存、信号量等。

所以，在选择是否使用 sync.Mutex 的时候可以考虑是否有其他方式代替。
