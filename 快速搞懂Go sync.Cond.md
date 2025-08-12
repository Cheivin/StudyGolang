# 快速搞懂Go sync.Cond

## 1. sync.Cond长啥样

```go
type Cond struct {
    noCopy noCopy           // 用于静态检查防止复制
    L    sync.Locker        // 外部锁（*Mutex / *RWMutex）
    notify notifyList       // 等待队列，FIFO
    checker copyChecker     // 用于运行时检查防止复制
}

// 它的结构和runtime/sema.go中的notifyList差不多
type notifyList struct {
    wait   uint32           // 已入队序号
    notify uint32           // 已唤醒序号
    head   *sudog           // 队列头
    tail   *sudog           // 队列尾
}
```

- `waitq` 是 单向链表，节点是 `sudog`（goroutine + 票号）
- **头尾指针** 保证 O(1) 入队 / 出队
- **无 prev 指针** → 省 8 字节 / 节点，GC 更快

## 2. 创建-等待-唤醒，分别是干什么？

- 有个游乐场，同一时间只能一个人进去游玩，大家排队得听管理员叫号。

1. 通过`NewCond`将手上的`锁(门票)`变成叫号器
    ```go
    mu := &sync.Mutex{}
    cond := sync.NewCond(mu)
    ```
    
1. 每个 `goroutine(玩家)`调用`cond.Wait()` 时，先 **交出手中的锁(门票)**，然后把自己包装成 **sudog(玩家信息)** 跑到等待队列尾部排队登记，接着调用 `gopark()`进入睡眠状态，等叫号。
    ```go
    cond.L.Lock()
    for !condition() {          // 防虚假唤醒
        cond.Wait()            // ① 解锁 ② 入队 ③ gopark()
    }
    cond.L.Unlock()
    ```

2. `管理员(调度器)`：负责管理等待队列，按 FIFO 顺序唤醒玩家。
    ```
    ┌----┐                 ┌----┐
    │  G │─Lock------------┤cond│
    └----┘                 └----┘
       │  condition? false
       │  Wait()
       ▼
    ┌--------┐       ┌--------┐
    │ unlock │------▶│ notifyQ│─tail→G→nil
    └--------┘       └--------┘
       ▲
       │被唤醒
       ▼
    ┌--------┐
    │ relock │
    └--------┘
    ```
    - **主动解锁**：防止死锁
    - **重新加锁**：醒来后再抢锁
    - **while 循环**：防虚假唤醒

3. 叫号入场
    1. `cond.Signal()`：管理员叫醒队列`头部`的玩家(head→goready())，该玩家醒来后重新竞争锁（重新买票入场）
    2. `cond.Broadcast()`：管理员一次性叫醒 所有 玩家(逐个 goready())，每个玩家醒来后重新竞争锁，按先后顺序入场。
    
    ```
    notifyList
     head→G1→G2→G3→nil
        ▲   ▲   ▲
        │   │   │
     Broadcast() 逐个 goready()
    ```
    - 被唤醒的 G 全部重新竞争同一把锁 **串行执行**
    - 未抢到锁的继续排队(由 `for` 循环决定是否再 Wait())
    
4. 玩家行为
    - **中途退场**：玩家可以在任何时候决定退出等待队列（通过 `break` 或 `context` 通知）。
    - **虚假唤醒**：管理员可能无故叫醒某个玩家，但此时游戏并未开始（条件未满足）。玩家需要重新检查条件，决定是否继续等待。

## 3. 面试题速查

#### 3.1 Cond 为什么必须配锁？

保护临界区 + 原子睡眠/唤醒。

#### 3.2 Wait 会释放锁吗？

会，先 `Unlock`，醒来再 `Lock`。

#### 3.3 Signal vs Broadcast？

`Signal` 唤醒 1 个，`Broadcast` 唤醒全部。

#### 3.4 虚假唤醒是什么，如何防御？

- 为什么会出现
    - **操作系统层面**：Linux futex、Windows condition variable 等底层同步原语，偶尔会 无理由唤醒 一个等待线程，以简化内核实现或避免丢失信号。
    - **Go 调度器层面**：调度器在 `goready()` 时可能一次性唤醒多个 G（例如 Broadcast），其中一部分 **条件已满足**，另一部分 **条件未满足**；后者就是虚假唤醒。
- 如何防御

    ```go
    cond.L.Lock()
    for !condition() {   // 必须 for，不是 if
        cond.Wait()
    }
    // 条件已满足，安全执行
    cond.L.Unlock()
    ```
    - **循环再检查**：醒来先确认条件，不满足继续睡。
    - **不会死循环**：条件满足后自然退出循环。

#### 3.5 忘记 for 会怎样？

只等一次，条件不满足就继续跑。

#### 3.6 能复制 Cond 吗？

不能，`noCopy`在`go vet`中检查并提示，`copyChecker`在运行时检查，触发`panic`

#### 3.7 忘记加锁会怎样？

死锁或竞态。

#### 3.8 Cond vs Channel？

- `Cond` 无数据拷贝，是条件变量，适合**条件满足时通知**场景和**高并发、低延迟**，比如连接池
- `Channel` 有数据拷贝，是数据流动，适合**数据传递**场景，比如生产-消费者。

#### 3.9 空 select vs Cond？

空 select 永久阻塞，Cond 可唤醒。
