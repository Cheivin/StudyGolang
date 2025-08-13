## 1. sync.WaitGroup 长啥样

```go
type WaitGroup struct {
    noCopy noCopy        // 静态防复制
    state  atomic.Uint64 // 高 32 位：counter；低 32 位：waiter
    sema   uint32        // runtime 信号量
}
```

- `sema` 是 `runtime` 用于挂起/唤醒的 `token`，上层无感知。
- `noCopy` 与 `sync`包其他结构中的定义一样，编译器拦住复制行为。
- `state` 只用 **一条 64 位原子变量** 存放两个 32 位整数，保证并发下既读 `counter` 又读 `waiter`，无需两把锁。

## 2. 实现原理

### 2.1 Add：计数器更新 + 归零广播

```go
state := wg.state.Add(uint64(delta) << 32) // 把 delta 塞进高 32 位
v := int32(state >> 32) // 高位取出 counter
w := uint32(state)      // 低位取出 waiter
if v < 0 {
	panic("sync: negative WaitGroup counter")
}
if w != 0 && delta > 0 && v == int32(delta) {
	panic("sync: WaitGroup misuse: Add called concurrently with Wait")
}
if v > 0 || w == 0 {
	return
}
if wg.state.Load() != state {
	panic("sync: WaitGroup misuse: Add called concurrently with Wait")
}
	wg.state.Store(0)
for ; w != 0; w-- {
	runtime_Semrelease(&wg.sema, false, 0)
}
```
1. `counter` 为负 → `panic`：避免逻辑错误。
2. `counter > 0` 或无 `waiter` → 直接返回：快速路径。
3. misuse 检测：
    1. `Add` 与 `Wait` 并发会触发 `panic`。
    2. 计数器归 0 后再次 `Add` 必须等所有 `Wait` 返回，否则 `panic`。
4. `counter == 0` 且 `waiter > 0` → 唤醒全部
    1. **广播唤醒**：一次性发 w 个信号，所有 `Wait` 在同一时刻被唤醒，相当于 `Broadcast`
    2. **唤醒顺序**：FIFO

### 2.2 Done：Add(-1)的语法糖
```go
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```
- 任何 `goroutine` 完成任务只需一句 `Done()`，更简洁且更语义化。

### 2.3 Wait：CAS 自旋 + 信号量阻塞
```go
for {
    state := wg.state.Load()
    v := int32(state >> 32)   // 高位取出 counter
    w := uint32(state)        // 低位取出 waiter
    if v == 0 { return }      // 快速路径：已归零
    if wg.state.CompareAndSwap(state, state+1) {  // CAS：把 waiter++
        runtime_SemacquireWaitGroup(sema)         // 阻塞
        if wg.state.Load() != 0 {                 // 连续调用检查
            panic("sync: WaitGroup is reused before previous Wait has returned") 
        }
        return
    }
}
```

- **无锁自旋**：`CAS` 失败后重试，避免全局锁。
- **一次阻塞**：成功后挂在 `runtime` 信号量上，`CPU` 不占时间片。
- **唤醒后校验**：再次读 `state`，非 0 说明 `WaitGroup` 被提前复用，直接 `panic`。

## 3. 与channel的对比
| 维度   | WaitGroup                              | Channel                           |
| ---- | -------------------------------------- | --------------------------------- |
| 通信模型 | 纯同步：只关心「还有多少个任务」                       | 数据流动：发送-接收携带值                     |
| 使用姿势 | `Add` → 启动 goroutine → `Done` → `Wait` | `ch := make(chan T)` → 生产/消费      |
| 阻塞方式 | 计数器归零一次性唤醒全部                           | 每写一次阻塞/唤醒一次                       |
| 内部原语 | 64 位原子 + runtime 信号量                   | 环形队列 + sudog 链表 + 调度器 gopark      |
| 典型场景 | 并发任务汇聚、批量等待结束                          | 生产-消费、流水线、事件通知                    |
| 超时支持 | 需配合 `context` 或 `select`               | `select` 原生支持 `case <-time.After` |
| 复用风险 | 禁止复制、禁止提前复用                            | 可复制，但关闭后不能再写                      |
| 性能特点 | 无锁、广播、极低开销                             | 每次传递值都有内存拷贝                       |

> 一句话：WaitGroup 是“计数器 + 广播”，Channel 是“管道 + 事件”。

## 4. 面试题速记

#### 4.1. Add 的参数可以为负吗？边界是多少？
可以为负，但 counter 不能 < 0，否则 panic。

#### 4.3. WaitGroup 可以复制吗？
不能。noCopy 会让 go vet 报警；复制后并发使用会导致数据竞争或 panic。

#### 4.4. WaitGroup 如何实现超时等待？
本身不支持，需要配合 context.WithTimeout 或 select + time.After。

#### 4.5. counter 已经为 0 时再 Add 正数会怎样？
- 如果此时仍有旧 `Wait` 未结束则 `panic`
- 如果没有`Wait`，则可以调用，但新的 `Wait` 会立即返回 `0`

#### 4.6. Add 可以在 goroutine 里调吗？
在 `goroutine` 里调用，可能出现 `Wait` 提前返回。

#### 4.7. WaitGroup 与 ErrGroup 的区别？
`ErrGroup` 在 `WaitGroup` 基础上增加了 **收集第一个错误** 和 `context` 取消传播。

#### 4.8. 为什么 WaitGroup 内部不用互斥锁？
64 位原子变量 + runtime 信号量已经足够，避免全局锁带来的调度开销。

#### 4.9. 多个 goroutine 同时 Wait 会发生什么？
全部阻塞；`counter` 归零后一次性唤醒，唤醒顺序由 `runtime` 信号量保证 `FIFO`。

#### 4.WaitGroup 适用于哪些场景？
并发请求聚合、批量 IO、Map-Reduce 阶段同步、测试用例等待后台 goroutine 结束等。

#### 4.WaitGroup 计数器归零后还能再次使用吗？
可以，但要确保所有 `Wait` 已经返回，且新的 `Add` 发生在旧 `Wait` 之后。