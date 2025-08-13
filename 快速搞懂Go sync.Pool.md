## 1. sync.Pool长啥样

### 1.1 Pool顶层结构

```go
type Pool struct {
    noCopy noCopy
    local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
    localSize uintptr        // size of the local array
    victim     unsafe.Pointer // local from previous cycle
    victimSize uintptr        // size of victims array
    New func() interface{}    // 自定义的对象创建回调函数
}
```

- **`local`**：指向一个数组，每个元素是一个 `poolLocal`，对应一个 P（处理器）。每个 P 有自己的本地缓存，减少锁竞争。
- **`localSize`**：`local` 数组的大小，通常等于 `GOMAXPROCS`。
- **`victim`**：上一轮 GC 期间的 `local`，用于在 GC 后平滑过渡。
- **`victimSize`**：`victim` 数组的大小。
- **`New`**：当池中没有可用对象时，调用此函数创建新对象。

### 1.2 poolLocal
每个 P 的本地缓存，包含私有和共享两部分：

```go
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte // 防止 false sharing
}

type poolLocalInternal struct {
    private interface{} // P 私有的单个对象
    shared poolChain    // 共享对象链表
}
```

- **`private`**：P 私有的对象，无锁访问。
- **`shared`**：共享对象链表，其他 P 可以窃取。

### 1.3 poolChain
共享对象链表，每个节点是一个环形队列：

```go
type poolChain struct {
    head *poolChainElt
    tail *poolChainElt
}

type poolChainElt struct {
    poolDequeue
    next, prev *poolChainElt
}

type poolDequeue struct {
    headTail uint64
    vals []eface
}
```

- **`poolDequeue`**：环形队列，单生产者多消费者。
- **`headTail`**：32 位 head 和 32 位 tail，通过原子操作更新。
- **`vals`**：存储对象的数组。

---

## 2. 实现原理

### 2.1. Get 流程

#### 2.1.1 先取本地缓存（`private`）
- **优先级最高**：直接从当前 P 的 `private` 字段获取对象。
- **无锁访问**：`private` 是当前 P 的私有字段，访问无需加锁。

#### 2.1.2 再找共享缓存（`shared`）
- **本地共享链表**：如果 `private` 为空，尝试从当前 P 的 `shared` 链表中获取对象。
- **窃取机制**：如果本地共享链表为空，尝试从其他 P 的共享链表中窃取对象。

#### 2.1.3 然后窃取其他 P 的共享缓存
- **全局窃取**：如果当前 P 的共享链表为空，尝试从其他 P 的共享链表中窃取对象。
- **随机选择**：随机选择其他 P 的共享链表，避免总是访问同一个 P。

#### 2.1.4 最后创建新对象
- **调用 `New` 函数**：如果所有缓存都为空，调用 `New` 函数创建新对象。
- **线程安全**：`New` 函数可以并发调用，但每次调用都独立创建一个新对象。

### 2.2. Put 流程

#### 2.2.1 先放入本地缓存（`private`）
- **优先级最高**：优先放入当前 P 的 `private` 字段。
- **无锁访问**：`private` 是当前 P 的私有字段，访问无需加锁。

#### 2.2 再放入共享缓存（`shared`）
- **本地共享链表**：如果 `private` 已满，尝试放入当前 P 的 `shared` 链表。
- **线程安全**：共享链表的访问需要加锁，确保线程安全。

### 2.3. GC 敏感性
- **GC 周期**：`sync.Pool` 的对象可能在两次 GC 之间被清理，因此不适合存储长期存活的对象。
- **平滑过渡**：在 GC 期间，`sync.Pool` 会将当前的 `local` 和 `victim` 交换，确保平滑过渡。

## 3. 性能优势
- **减少内存分配**：通过复用对象，减少频繁创建和销毁对象的开销。
- **减少 GC 压力**：对象复用减少了 GC 的工作量。
- **线程局部存储**：每个 P 有自己的本地缓存，减少锁竞争。

## 4. 面试题速查

#### 4.1. `sync.Pool` 如何减少锁竞争？
使用 **线程局部存储**（每个 P 有自己的缓存），减少全局锁竞争。

#### 4.2. `sync.Pool` 是否线程安全？
线程安全，内部使用锁和线程局部存储。

#### 4.3. `sync.Pool` 是否支持并发 `Get` 和 `Put`？
支持，内部机制保证并发安全。

#### 4.4. `sync.Pool` 的对象何时被回收？
在 **线程局部存储已满** 或 **GC 时**，对象被回收。

#### 4.5. `sync.Pool` 和 `sync.Map` 的区别是什么？
`sync.Pool` 用于缓存对象，`sync.Map` 用于线程安全的键值存储。

#### 4.6. `sync.Pool` 的性能优势是什么？
减少内存分配和 GC 压力，适合高并发场景。

#### 4.7. `sync.Pool` 是否适合所有场景？
适合 **高并发、对象复用** 的场景，不适合 **长时间持有对象** 的场景。

#### 4.8. `sync.Pool` 的 `New` 函数何时被调用？
当缓存中没有对象时，`New` 函数被调用。

#### 4.9. `sync.Pool` 的对象是否可以是任意类型？
可以，但需要通过接口类型转换。

#### 4.10. `sync.Pool` 的 `victim` 字段有什么作用？
- **答案**：用于 **垃圾回收**，回收被清理的线程局部存储。

#### 4.11. `sync.Pool` 是否支持泛型？
- **答案**：不支持，`sync.Pool` 是基于接口的，不涉及值存储。

#### 4.12. `sync.Pool` 的对象是否可以是 nil？
可以，但需要在 `Get` 时检查。

## 5. 补充

### 5.1 False Sharing 与 `pad` 字段

#### 5.1.1. False Sharing 是什么？

**False sharing** 指的是多个线程访问内存中相邻的缓存行(cache line)，导致不必要的缓存失效和性能下降。有点类似于cache line抖动，但cache line抖动是因为多线程对同一line整行写导致相互踢出，而false sharing只是写同一line的小部分，导致整行失效。

#### 5.1.2. `pad` 字段的作用

`pad` 字段的作用是 **防止 false sharing**，通过在 `poolLocal` 结构体中插入额外的字节，确保 `poolLocal` 的大小是缓存行大小（128 字节）的倍数，从而避免不同 `poolLocal` 实例之间的缓存行冲突。

```go
type poolLocal struct {
    poolLocalInternal
    pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte // 防止 false sharing
}
```

- **`poolLocalInternal`**：包含 `private` 和 `shared` 字段。
- **`pad`**：填充字节，确保 `poolLocal` 的大小是 128 字节的倍数。

#### 5.1.3 为什么是 128 字节？

- 在 x86-64 架构中，缓存行大小通常是 64 字节。
- 为了确保 `poolLocal` 的大小是缓存行大小的倍数，`sync.Pool` 选择了一个更大的值（128 字节），这样可以更有效地防止 false sharing。

#### 5.1.4 防止 false sharing 的效果

- 每个 P（处理器）都有自己的 `poolLocal` 实例。
- 通过填充 `pad`，确保不同 P 的 `poolLocal` 实例不会共享同一个缓存行。
- 这样，一个 P 的操作不会影响其他 P 的缓存行，从而减少缓存失效和性能下降。